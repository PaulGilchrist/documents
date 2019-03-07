# Angular VSTS Build Steps to Leverage Environment Files, and Test Scripts

## Azure Setup

1. Create a new Azure App Service – Web App with the following AppSettings
   * Platform = "64-bit"
   * HTTP Version = "2.0"
   * Affinity = "Off"

## Build Tasks

1. Add Node install step: Node Tool Installer
   * Set to version spec 8.9 (required for Angular Cli 6)
2. Add pakcage install step: NPM
   * Command = install
3. Optional - Add unit test step: NPM
   * Display Name = test
   * Command = custom
   * Command and arguments = `run ng -- test`
4. Add build step: NPM  (`build` script must exist in `package.json`)
   * Display Name = build
   * Command = custom
   * Command and arguments
      * Development = `run ng -- build`
      * QA (if setup in project) = `run ng -- build -c qa`
      * Staging (if setup in project) = `run ng -- build -c staging`
      * Production = `run ng -- build -c production --prod`
5. Add deploy step: `Azure App Service Deploy` or optionally `Publish Artifact: Site`
   * Package or folder = `$(System.DefaultWorkingDirectory)/dist`
   * If project subfolder was not previously removed from angular.json dist folder in (above step), then it must be added here
      * example: `$(System.DefaultWorkingDirectory)/dist/<angularProjectName>`
   * Check `Additional Deployment Options/Publish using Web Deploy`
   * Check `Additional Deployment Options/Remove additional files at destination`
6. Save & queue
7. Confirm successful deployment and smoke test application URL
