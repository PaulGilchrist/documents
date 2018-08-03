# Angular Module and Folder Structure Recommendations

An Angular application should be separated into modules with as little code as necessary placed in the root module.  Each module will be contained in its own folder and identified as a module through the naming convention `<name>-module`.  Within the module’s root folder would be the module definition (`name-module.ts`) and the following optional sub-folders:

* components
* data
* directives
* models
* pipes
* services

The module may also contain additional child modules each self-contained similar to its parent module.  WebPack separates each module into its own JS file and when configured for lazy loading, does not send that file to the client until a feature within the module is requested.  This ensure quicker application startup, and better module portability.

The `app.module` should contain as little as possible, and is usually limited to components or services needed across the application like the `app.component`, main navigation menu, and user identity service.  Other components or services that are optionally used across modules should be included in a `shared` module, sometimes also referred to as a `common` module.  The `shared-module` folder would be a sub-folder of the `app` folder like as described above for child modules.

Within any components folder would be separate folders for each component.  This separation is due to each component usually being made up of multiple files such as class, spec, HTML, and styles files.

Jasmine unit test spec files are contained in the same folder as the code being tested meeting with Angular best practice, and keeping the component or service more portable.  Angular will automatically discover all spec files and include them in unit testing when running `ng test`
