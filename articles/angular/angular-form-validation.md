# Form Validation

Form validation is required to prevent web form abuse by malicious users. Improper validation of form data is one of the main causes of security exploits such as header injections, cross-site scripting, SQL injections, buffer overflows, and more.  Following the below recommendations will go a long way towards reducing these types of exploits. 

## Recommendations

* Clearly mark and enforce required fields

```html
<input type="text" #nameControl="ngModel" required>
<div [hidden]="nameControl.valid || nameControl.pristine" class="alert alert-danger">
    Name is required
</div>
```

* All strings should enforce maximum lengths, and optionally minimum lengths

```html
<input type="text" #nameControl="ngModel"  minlength="2" maxlength="20">
<div [hidden]="nameControl.valid || nameControl.pristine" class="alert alert-danger">
    Name must be between 2 and 20 characters
</div>
```

* All numbers should enforce minimum and maximum values

```html
<input type="number" #ratingControl="ngModel"  min="1" max="5">
<div [hidden]="ratingControl.valid || ratingControl.pristine" class="alert alert-danger">
    Rating must be between 1 and 5
</div>
```

* Leverage HTML5 input types (email,url, number, tel, date, password) – RegEx “pattern” may still be needed

```html
<input type="tel" #phoneControl="ngModel"  pattern='[\(]\d{3}[\)][\s]\d{3}[\-]\d{4}'>
<div [hidden]="phoneControl.valid || phoneControl.pristine" class="alert alert-danger">
    Phone is not in the required format (###) ###-####
</div>
```

* Use placeholders to help guide the user in proper format

```html
<input type="tel" placeholder="(###) ###-####">
```

* Use pick lists where possible to prevent misspelling or case differences

```html
<select name="state" [ngModel]="state" (change)="onUpdateState($event)" #stateControl="ngModel">
    <option *ngFor="let s of states" [value]='s.abbreviation'>{{s.name}}</option>
</select>
```

* Error messages should be easy to understand and actionable
* Error message should be near the field and clear which field it refers to

```html
<div class="form-group" [class.has-error]="firstNameControl.invalid && firstNameControl.dirty">
    <label class="control-label">First Name</label>
    <div class="input-group margin-bottom-sm">
        <div class="input-group-prepend">
            <div class="input-group-text"><i class="fa fa-user fa-fw"></i></div>
            </div>
            <input type="text" name="firstName" [(ngModel)]="firstName" #firstNameControl="ngModel" class="form-control"
                    placeholder="First Name" pattern="[A-Za-z ,.'-]+" required autofocus minlength="2" maxlength="20">
        </div>
        <div [hidden]="firstNameControl.valid || firstNameControl.pristine" class="alert alert-danger">
            First name is required and must be between 2 and 20 characters
        </div>
    </div>
```

* Use dirty/touched checks to not show errors unless the field has been changed

```html
<div [hidden]="control.valid || control.pristine" class="alert alert-danger">
    Error goes here and only displays if validity checks fail
</div>
```

* Don’t use dialog boxes, popups, or error pages
* Use icons, colors, borders, or other visual cues to bring attention to errors
* Use real time feedback rather than validating the whole form only on save/submit
* Disable form save/submit until all fields are valid

```html
<button class="btn btn-success" [disabled]="userForm.pristine || userForm.invalid" (click)="saveForm()">
    <i class="fa fa-check"></i> Save
</button>
```

Client-side validation does not eliminate the need to also do server-side data and business rule validation.

## Appendix – RegEx Expressions

```cs
Date = "\d{1,2}/\d{1,2}/\d{4}"  
Email = "/^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]+)*$/"  
IPv4 = "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"  
Latitude/Longitude = "-?\d{1,3}\.\d+"  
Phone = "[0-9]{3}-[0-9]{3}-[0-9]{4}"  
Price = "\d+(\.\d{2})?"  
Url = "https?://.+"  
Zip = "^\d{5}-\d{4}|\d{5}|[A-Z]\d[A-Z] \d[A-Z]\d$"  
```
