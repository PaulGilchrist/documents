# Draft Template - Upgrading from Angular 5 to Angular 6

* Discuss change from angular-cli.json to angular.json with internal structure changes
  * `ng update @angular/cli --migrate-only --from=1.7.3`
* Discuss change from `@angular/http` to `@angular/common/http`
  * Discuss change from map() to pipe(), retry(), tap(), and catchError()