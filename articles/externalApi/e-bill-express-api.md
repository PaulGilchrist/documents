# Wells Fargo e-Bill Express API Integration

This document will summarize the steps for using Wells Fargo's e-Bill Express API for submitting an invoice, and retreiving payment.

## Steps

1. Retrieve the e-Bill Express RSA key from `appsettings.json` or an environment variable

2. Setup invoice model

```cs
    public class Invoice {
        public decimal AmountDue { get; set; }
        public string InvoiceNumber { get; set; }
        public DateTime DueDate { get; set; }
        public string FDICode { get; set; }
        public string LastName { get; set; }
        public string SalesAgreementNumber { get; set; }
        public string CustomerName { get; set; }
        public string CommunityNumber { get; set; }
        public string LotBlock { get; set; }
        public string DepositType { get; set; }
        public int ID { get; set; }
    }
```

3. Setup RSA functions needed for decoding the e_Bill Express RSA Key

```cs
private static RSACryptoServiceProvider DecodeRSAPrivateKey(byte[] key) {
    byte[] MODULUS, E, D, P, Q, DP, DQ, IQ;
    // --------- Set up stream to decode the asn.1 encoded RSA private key ------
    MemoryStream mem = new MemoryStream(key);
    BinaryReader binr = new BinaryReader(mem);  //wrap Memory Stream with BinaryReader for easy reading
    byte bt = 0;
    ushort twobytes = 0;
    int elems = 0;
    try {
        twobytes = binr.ReadUInt16();
        if (twobytes == 0x8130) //data read as little endian order (actual data order for Sequence is 30 81)
            binr.ReadByte();    //advance 1 byte
        else if (twobytes == 0x8230)
            binr.ReadInt16();    //advance 2 bytes
        else
            return null;
        twobytes = binr.ReadUInt16();
        if (twobytes != 0x0102) //version number
            return null;
        bt = binr.ReadByte();
        if (bt != 0x00)
            return null;
        //------ all private key components are Integer sequences ----
        elems = GetIntegerSize(binr);
        MODULUS = binr.ReadBytes(elems);
        elems = GetIntegerSize(binr);
        E = binr.ReadBytes(elems);
        elems = GetIntegerSize(binr);
        D = binr.ReadBytes(elems);
        elems = GetIntegerSize(binr);
        P = binr.ReadBytes(elems);
        elems = GetIntegerSize(binr);
        Q = binr.ReadBytes(elems);
        elems = GetIntegerSize(binr);
        DP = binr.ReadBytes(elems);
        elems = GetIntegerSize(binr);
        DQ = binr.ReadBytes(elems);
        elems = GetIntegerSize(binr);
        IQ = binr.ReadBytes(elems);
        // ------- create RSACryptoServiceProvider instance and initialize with public key -----
        CspParameters cp = new CspParameters();
        RSACryptoServiceProvider rsa = new RSACryptoServiceProvider(1024, cp);
        RSAParameters rsaParams = new RSAParameters();
        rsaParams.Modulus = MODULUS;
        rsaParams.Exponent = E;
        rsaParams.D = D;
        rsaParams.P = P;
        rsaParams.Q = Q;
        rsaParams.DP = DP;
        rsaParams.DQ = DQ;
        rsaParams.InverseQ = IQ;
        rsa.ImportParameters(rsaParams);
        return rsa;
    } catch (Exception ex) {
        return null;
    } finally {
        binr.Close();
    }
}

private static int GetIntegerSize(BinaryReader binr) {
    byte bt = 0;
    byte lowbyte = 0x00;
    byte highbyte = 0x00;
    int count = 0;
    bt = binr.ReadByte();
    if (bt != 0x02)     //expect integer
        return 0;
    bt = binr.ReadByte();
    if (bt == 0x81)
        count = binr.ReadByte();    // data size in next byte
    else if (bt == 0x82) {
        highbyte = binr.ReadByte(); // data size in next 2 bytes
        lowbyte = binr.ReadByte();
        byte[] modint = { lowbyte, highbyte, 0x00, 0x00 };
        count = BitConverter.ToInt32(modint, 0);
    } else {
        count = bt;     // we already have the data size
    }
    while (binr.ReadByte() == 0x00) {   //remove high order zeros in data
        count -= 1;
    }
    binr.BaseStream.Seek(-1, SeekOrigin.Current);       //last ReadByte wasn't a removed zero, so back up a byte
    return count;
}
```

4. Setup class with ability to authenticate with e-Bill Express and create invoices

```cs
public class EBillExpressService {
    private IMemoryCache _cache;
    private IConfiguration _config;
    private string _apiUrl;
    private RSACryptoServiceProvider _privateKey;
    public EBillExpressService(IMemoryCache cache, IConfiguration config) {
        _cache = cache;
        _config = config;
        _apiUrl = config.GetValue<string>("EBillExpress:ApiUrl");
        var pem = Convert.FromBase64String(_config.GetValue<string>("EBillExpress:RSAKey"));
        _privateKey = DecodeRSAPrivateKey(pem);
    }

    private async Task<string> GetAccessToken() {
        var accessToken = _cache.Get<string>("EBillAccessToken");
        if (accessToken == null) {
            var secret = _config.GetValue<string>("EBillExpress:Secret");
            var dateTime = DateTime.Now.ToString("o");
            var msg = string.Concat(secret, dateTime);
            byte[] msgBytes = Encoding.UTF8.GetBytes(msg);
            var signatureBytes = _privateKey.SignData(msgBytes, "SHA512");
            var signature = Convert.ToBase64String(signatureBytes);
            var authRq = new {
                APIKey = _config.GetValue<string>("EBillExpress:ApiKey"),
                CreationDate = dateTime,
                Signature = signature,
                grant_type = "client_credentials",
                Role = "APIEBPPBillerCSR"
            };
            var client = new WebClient();
            client.Headers.Add(HttpRequestHeader.Accept, "application/json");
            client.Headers.Add(HttpRequestHeader.ContentType, "application/json;charset=UTF-8");
            var rs = JObject.Parse(await client.UploadStringTaskAsync(string.Format("{0}/api/Auth", _apiUrl), "POST", JsonConvert.SerializeObject(authRq)));
            accessToken = rs.Value<string>("access_token");
            _cache.Set("EBillAccessToken", accessToken, TimeSpan.FromMinutes(59));
        }
        return accessToken;
    }

    public async Task CreateInvoice(Invoice invoice) {
        var accessToken = await GetAccessToken();
        var rq = new {
            Invoices = new[] {
                new {
                    AmountDue = invoice.AmountDue,
                    BillerInvoiceNo = invoice.InvoiceNumber,
                    BillerRemittanceField1 = invoice.CommunityNumber,
                    BillerRemittanceField2 = invoice.LotBlock,
                    BillerRemittanceField3 = invoice.DepositType,
                    CompanyName = invoice.CustomerName,
                    DueDate = invoice.DueDate,
                    FDICode = invoice.FDICode,
                    OtherData = invoice.LastName,
                    ReferenceNumber = invoice.SalesAgreementNumber,
                    Status = "Active"
                }
            }
        };
        var client = new WebClient();
        client.Headers.Add(HttpRequestHeader.Accept, "application/json");
        client.Headers.Add(HttpRequestHeader.ContentType, "application/json;charset=UTF-8");
        client.Headers.Add(HttpRequestHeader.Authorization, $"Bearer {accessToken}");
        var rs = JObject.Parse(await client.UploadStringTaskAsync(string.Format("{0}/api/Invoice", _apiUrl), "PUT", JsonConvert.SerializeObject(rq)));
        if (!rs.Value<bool>("Success")) {
            JArray err = rs.GetValue("Errors") as JArray;
            throw new Exception(string.Join(", ", err.Select(tok => tok.Value<string>()).ToList()));
        }
    }
}
```

5. Inform customer of `InvoiceNumber` and then HTTP redirect them to e-Bill Express webiste for them to pay invoice using either credit card or ACH bank transfer.

```html
<a href="{{ eBillUrl }}" target="_blank">Go to E-Bill Express</a>
```