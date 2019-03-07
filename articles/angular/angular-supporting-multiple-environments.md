# Angular Supporting Multiple Environments

Creating a new Angular project using Angular CLI’s ‘ng new’ will create both non-production and production environment files, and configure optimizations when compiling and bundling for production.  The below steps can be used to add additional environments like QA, Stating, Training, etc. to the project.

## Steps

1. Locate existing environment files at `src/environments`.  The file named `environment.ts` will be the default when not explicitly selected during the build process
2. Create one file for each additional environment naming them `environment.<environmentName>.ts`.  Ex: `environment.qa.ts`, `environment.staging.ts`, etc.
3. Add environment specific variables to each file, keeping their keys and structure consistent, but varying their values to match the corresponding environment.  Example:

```javascript
export const environment = {
	apiUrl: 'https://apiDev.azurewebsites.net',
	azureAuthProvider: {
		'aadInstance': 'https://login.microsoftonline.com/{0}',
		'clientId': 'bd065891-b008-4968-9b26-5f2bcb9c1b66',
		'tenant': 'myCompany.onmicrosoft.com'
	},
	envName: 'development',
	production: false
};
```

4. Add each new environment to `angular.json` in section `projects/architect/configurations`.  You can use the production environment as a guide, but remember to change the `fileReplacements` `'with'` name to match the new environment’s file name, and remove any optimizations that are not required for the given environment.  As examples, for QA, this may mean no optimizations, where for staging it may mean a close match to production.
5. Add each new environment to `angular.json` in section `projects/architect/serve/configurations`.  You can use the production environment as a guide, but remember to change the `browserTarget` name to match the new environment’s name.  This will be the `--configuration` name passed during an `ng serve` or `ng build` command
