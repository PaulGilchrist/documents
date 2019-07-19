# Angular - Adding Bootstrap 4

Bootstrap 4 was developed using SASS, and although it can be deployed to a CSS only project, to get the best benefits, including variable override capabilites, the project should be enhanced to support both SASS and CSS using the below steps when adding Bootstrap

1. Add `bootstrap` to `project.json`, and `npm install` it into the project

2. Edit file `angular.json`, in the `project, schematics` section add the following:

```json
"@schematics/angular:component": {
    "styleext": "scss"
}
```

3. Edit file `angular.json`, in the `project, architect, options` section changing the styles file extention to `scss`
4. Edit file `angular.json`, in the `project, architect, options` section add the following:

```json
"stylePreprocessorOptions": {
   "includePaths": [
       "src"
   ]
},
"scripts": [
    "node_modules/bootstrap/dist/js/bootstrap.js"
]
```

5. Within file `angular.json`, duplicate the above steps for the test project
6. Rename styles.css to `styles.scss`
7. import bootstrap into `styles.scss` by adding the line `@import '~bootstrap/scss/bootstrap.scss';`

## Conversion from Bootstrap 3 to 4

* Project should use SCSS to control variables like colors
* Replace panels with cards
* Collapse simplified to `[class.collapse]="!isOpen"` setting this variable in code or by a `(click)` or similar action
* Replace `hidden-<size>` & `visible-<size>` with combination of `d-<size>-none` and `d-<size>-<displayType>`
* Replace `pull-right`, `pull-left` with `float-right`, `float-left`
   * Better to use `d-flex` and `ml-auto` or `mr-auto` to fill middle
* Use `input-group-prepend` and `append` to get buttons or icons within an input