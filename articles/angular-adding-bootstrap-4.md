# Angular - Adding Bootstrap 4

Bootstrap 4 was developed using SASS, and although it can be deployed to a CSS only project, to get the best benefits, including variable override capabilites, the project should be enhanced to support both SASS and CSS using the below steps when adding Bootstrap

1. Edit file `angular.json`, in the `project, schematics` section add the following:

```json
"@schematics/angular:component": {
    "styleext": "scss"
}
```

2. Edit file `angular.json`, in the `project, architect, options` section changing the styles file extention to `scss`
3. Edit file `angular.json`, in the `project, architect, options` section add the following:

```json
"stylePreprocessorOptions": {
   "includePaths": [
       "src"
   ]
},
```

4. Within file `angular.json`, duplicate the above steps for the test project
5. Rename styles.css to `styles.scss`
6. import bootstrap into `styles.scss` by adding the line `@import '~bootstrap/scss/bootstrap.scss';`
