# Client Tools Setup and Update Guide for Front-End Development

This document will discuss the steps for setup or updating of all tools recommended for front-end web development when using the Angular framework.  Some back-end, testing,a nd other options tools will also be covered.

## Steps

1. Optional - Download and install/update the latest version of `Chrome` from [google.com](https://www.google.com/chrome/)
   * Chrome will be used for it client side debugging tools

2. Download and install/update the latest version of Node from [nodejs.org](https://nodejs.org/en/download/)
   * Accept all the defaults during the setup process
   * If Node is already installed, you can compare the current local version against the downloadable version using the commend `node -v`

3. Install or update the following global Node `packages`

```cmd
npm i -g npm
npm i -g @angular/cli
npm i -g jasmine
npm i -g karma
npm i -g newman
npm i -g protractor
npm i -g rxjs-tslint
npm i -g tslint
npm i -g typescript
npm i -g webpack
```

4. Download and install the latest version of `Visual Studio Code` (VS Code) from [visualstudio.com](https://code.visualstudio.com/download) even if you plan to use Visual Studio as your IDE, and VS Code will still be useful as GIT's default editor.
   * After the initial installation, VS Code will auto-update as newer versions are released.
   * The following VS Code extensions are recommeded
      * Angular Language Service
      * Git Blame
      * Git History
      * EditorConfig for VS Code
      * Beautify
   * If using Azure, the following VS Code extensions are recommeded
      * Azure Account
      * Azure Repos
      * Azure Storage
   * If using Docker, the following VS Code extensions are recommeded
      * Docker
      * Kubernetes
   * If using C# or otherwise planning to debug server side rather than client side, the following VS Code extensions are recommeded
      * C#
      * Debugger for Chrome

5. Download and install/update the latest version of `GIT` from [git-scm.com](https://git-scm.com/downloads)
   * If GIT is already installed, you can compare the current local version against the downloadable version using the commend `git --version`.  After the initial installation, GIT will auto-update as newer versions are released.
   * During installation, in the section titled `Choosing the default editor used by Git`, choose `Use Visual Studio Code as Git's default editor`
   * During installation, in the section titled `Adjusting your PATH environment`, choose `Git from the command line and also from 3rd-party software`
   * During installation, in the section titled `Choosing HTTPS transport backend`, choose `Use the OpenSSL library`
   * During installation, in the section titled `Configuring the line ending conversions`, choose `Checkout Widnows-style, commit Unix-style line endings`
   * During installation, in the section titled `Configuring the terminal emulator to use the Git Bash`, choose `Use Windows' default console window`
   * During installation, in the section titled `Configuring extra options`, select all possible options
   * During installation, in the section titled `Configuring experimental options`, do not select any options

6. Download and install/update the latest version of `Postman` from [getpostman.com](https://www.getpostman.com/downloads/)
   * After the initial installation, Postman will auto-update as newer versions are released.

7. Optionally download and install/update the latest version of `Notepad++` from [notepad-plus-plus.org](https://notepad-plus-plus.org/downloads/)

8. If developed application will run within a container, install docker using the following steps:
   * Use `Windows` key then search and run `Turn Windows features on or off` selecting both `Containers` and `Hyper-V`.  Reboot if requested during setup.
   * Download and install/update the latest version of `Docker` from [docker.com](https://www.docker.com/products/docker-desktop)
   * After the initial installation, Docker will auto-update as newer versions are released.
   * Once Docker is running in the `taskbar`, right click it, select `settings`, select `Kubernetes`, then select the checkbox titled `Enable Kubernetes`.

9. If doing SQL development or testing, download and install the latest version of `SQL Server Management Studio` (SSMS) from [microsoft.com](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms)
