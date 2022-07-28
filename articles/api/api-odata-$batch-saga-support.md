# OData $batch SAGA Support


When an OData $batch contains a ChangeSet, every change within that set must succeed or every change must fail.  This is equally true for change notification messages, where no messages can be sent until all changes are successful, then all notifications must be sent together as their own set maintaining proper sequencing.  To support them, the message service must support immediate sending of event messages not part of a changeset, and delayed messages when part of a changeset.  The delayed messages would queue up and all be sent when the changeset if completed successfully, or never sent if the changeset fails.

Below is example code showing a message service base class that has support for delayed messages:

```cs
using System.Diagnostics;
using API.Models;

namespace API.Services {
    public abstract class MessageService: IMessageService {
        protected readonly ApplicationSettings _applicationSettings;
        protected List<EventMessage> _eventMessages;
        protected bool disposedValue;

        public MessageService(ApplicationSettings applicationSettings) {
            _applicationSettings = applicationSettings;
            _eventMessages = new List<EventMessage>();
        }

        public void ClearDelayed(Guid? changeSetId) {
            // Remove messages with the matching ChangeSetId
            _eventMessages.RemoveAll(message => message.ChangeSetId == changeSetId);
       }

        // Send message now or save the message to send later (delaySend)
        public abstract void Send(string queueName, string type, object? jsonSerializableData, Type? dataSerializableType, Guid? changeSetId = null);

        public void SendDelayed(Guid? changeSetId) {
            // Send delayed event messages with the matching ChangeSetId (send in FIFO order)
            try {
                _eventMessages.FindAll(message => message.ChangeSetId == changeSetId).ForEach(message => {
                    Send(message.QueueName, message.Type, message.JsonSerializableData, message.DataSerializableType, null);
                });
                ClearDelayed(changeSetId);
            } catch(Exception ex) {
                Activity.Current?.AddTag("exception",ex);
                throw;
            }
        }

        public void Dispose() {
            // Do not change this code. Put cleanup code in 'Dispose(bool disposing)' method
            Dispose(disposing: true);
            GC.SuppressFinalize(this);
        }

        protected virtual void Dispose(bool disposing) {
            if(!disposedValue) {
                if(disposing) {
                    // TODO: dispose managed state (managed objects)
                }
                // TODO: free unmanaged resources (unmanaged objects) and override finalizer
                // TODO: set large fields to null
                disposedValue = true;
            }
        }

    }
}
```

The next step is to create an `ODataBatchhandler` class to manage the database and message success or failure:

```cs
using API.Services;
using Microsoft.AspNetCore.OData.Abstracts;
using Microsoft.AspNetCore.OData.Batch;
using Microsoft.AspNetCore.OData.Extensions;

namespace API.Classes {
    public class MyODataBatchHandler : DefaultODataBatchHandler {

        private readonly IMessageService _messageService;

        public MyODataBatchHandler(IMessageService messageService) : base() {
            _messageService = messageService;
        }

        public static Guid? GetChangeSetId(HttpRequest request) {
            return ((ODataBatchFeature?)request.HttpContext.Features[typeof(IODataBatchFeature)])?.ChangeSetId;
        }

        public override async Task<IList<ODataBatchRequestItem>> ParseBatchRequestsAsync(HttpContext context) {
            var requests = await base.ParseBatchRequestsAsync(context);
            return requests.Select(rq => {
                if (rq is ChangeSetRequestItem) {
                    return new TransactionalChangesetRequestItem(rq as ChangeSetRequestItem, _messageService);
                } else {
                    return rq;
                }
            }).ToList();
        }

        public class TransactionalChangesetRequestItem : ODataBatchRequestItem {
            private ChangeSetRequestItem _changeSetRequestItem;
            private readonly IMessageService _messageService;

            public TransactionalChangesetRequestItem(ChangeSetRequestItem changeSetRequestItem, IMessageService messageService) : base() {
                _changeSetRequestItem = changeSetRequestItem;
                _messageService = messageService;
            }

            public override async Task<ODataBatchResponseItem> SendRequestAsync(RequestDelegate handler) {
                var response = await _changeSetRequestItem.SendRequestAsync(handler) as ChangeSetResponseItem;
                var changeSetId = GetChangeSetId(_changeSetRequestItem.Contexts.First().Request);
                if (response != null && response.Contexts.All(c => c.Response.IsSuccessStatusCode())) {
                    // Write all events and commit changes to DB if successful
                    Console.WriteLine("Sending all events");
                    try {
                        _messageService.SendDelayed(changeSetId);
                        //transaction.Commit();
                    } catch (Exception ex) {
                        //transaction.Rollback();
                        _messageService.ClearDelayed(changeSetId);
                        throw;
                    }
                } else {
                    //transaction.Rollback();
                    _messageService.ClearDelayed(changeSetId);
                }
                return response;
            }
        }
    }
}
```

Finally, the batch handler will have to be used in place of the default batch handler.  This can be done in the `Program.cs` file as follows:

```cs
builder.Services.AddControllers().AddOData((options, serviceProvider) => {
    var messageService = serviceProvider.GetService<IMessageService>();
    var odataBatchHandler = new MyODataBatchHandler(messageService);
    // ...
});
```