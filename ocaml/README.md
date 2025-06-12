# OCAML Cheatsheet

This cheatsheet is such that anyone who reads this would be able to setup a new OCAML project using `dune` without having to go through the hassle of reading multiple documentation that may or may not be updated and not cohesive (OCAML documentation is not tbe best).

This cheatsheet applies to Linux users, I don't use Windows :)

## 1. Installing OCAML

### 1.1. Setting up opam

1. Install `opam` (OCAML Package Manager) via your package manager
    For Arch users, `sudo pacman -S opam` or `yay -S opam`.

2. Initialise `opam` by running the command `opam init -y`.

3. Subsequently run `eval $(opam env)`. 
    This will initialise the environment to the context of the current switch. 
    This command will need to be run everytime you open a new shell or change switches.

Official documentation link: https://ocaml.org/docs/installing-ocaml

### 1.2. What is a switch?

A switch in the context of ocaml is basically like `virtualenv` in Python. It is a isolated ocaml environment. By default, the global switch will be created when `opam` is installed. However, I highly recommend you create a local one for each ocaml project.

Official documentation link: https://ocaml.org/docs/opam-switch-introduction

## 2. Setting up dune

### 2.1. Installing dune

Run the following command: `opam install dune`.

This will install dune to the default global switch.

### 2.2. What is dune?

It can be confusing as to what is purpose of `dune` and `opam` from the documentation since they have similar commands and seems to share very similar responsibilities.

Firstly, `dune` is the tool used to create a project, build a project, execute a project, run tests and manage the dependencies of that project.

Now, the management of dependencies in `dune` does not mean downloading and installing ocaml packages in your current project folder. It only means making sure versions of dependencies can be specified and locked.

Even though `opam` is the package manager to install ocaml packages into your switch(environment) of your project, do not use it to manage dependencies of your project. It is only used to manually install the dependencies that `dune` requires to build.

In order to install ocaml packages for `dune` to build the project, you must install the dependency via `opam install <package>`.

For example, if you need the dependency `cmdliner` (a library to build CLIs), you firstly have to `opam install cmdliner` and subsequently edit the `dune-project` file to add `cmdliner` as a a library dependency. Refer to the section on **Managing Dependencies** for more details.

Honestly, this would be a lot less confusing if ocaml just had `opam` to do what `dune` does, just like how Rust's `cargo` works, much easier and frictionless.

## 3. Setting up local project

### 3.1. Create a new project

Now that `dune` is installed in the default switch, we can use `dune` to create a new project.

Run the following command: `dune init proj project_name`

### 3.2. Create a local switch in project

Ensure you are in the root directory of your project

Run the following command: `opam switch create . <ocaml-compiler-version>`

This will create local switch in your project by creating the `_opam` directory in the root of your project folder. The `_opam` folder will contain all files of dependencies used in this local switch.

There is no need for an explicit command to switch from the default switch to the local one, `opam` automatically detects the local switch if it exists within a project.

Official documentation link: https://ocaml.org/docs/opam-switch-introduction

## 4. Managing dependencies

### 4.1. Adding third party dependencies

1. Install the third party dependency via `opam install <package>`. If you need to specify the version, run `opam install <package_name>.<version>`.
2. Edit the `dune-project` file. This file is similar to `package.json` in Node.js. 
We need to add a section called `depends`. I will admit the syntax is odd, as `dune` uses `S-exp` for their config files.

    Below shows how to add the ocaml dependency of version 5.3. **Note: Do not mess with the spaces, as that can cause `dune` to fail to parse this file and fail to build the project**
    The `5.3` can be changed to some other version number.

    ```
    (depends
      (ocaml (>= 5.3)))
    ```

    If we wanted to add another dependency, in this example, `cmdliner`, then:

    ```
    (depends 
      (ocaml (>= 5.3))
      (cmdliner (>= 1.3)))

    ```

    In general, you want to follow this syntax when adding a dependency:
    ```
    (depends
      (<dependency_name><SPACE>(>=<SPACE><version_number>))
      (<dependency_name><SPACE>(>=<SPACE><version_number>)))
    ```
    **Ensure that the last row always has the last right bracket to close the `depends` section**

3. Edit the `dune` files

    In your `./lib` directory, there would be a `dune` file. It looks something like this:

    ```
    (library
      (name <project_name>))

    ```
    Your `./bin` directory will have a `dune` file as well

    ```
    (executable
      (public_name <project_name>)
      (name main))
    ```
    Depending on where you need to use the dependency, you would have to add the `libraries` section to either one or both `dune` files via the following syntax:

    For `./lib/dune`
    ```
    (library
      (name project_name)
      (libraries <some_dependency_1> <some_dependency_2> <some_dependency_3>))
    ```

    For `./bin/dune`
    ```
    (executable
      (public_name <project_name>)
      (name main)
      (libraries <some_dependency_1> <some_dependency_2> <some_dependency_3>))
    ```

    For example, if we have 2 dependencies we installed, `cmdliner` and `fmt`, then `./lib/dune` would be:
    ```
    (library
      (name project_name)
      (libraries cmdliner fmt))
    ```

    and `./bin/dune` would be:
    ```
    (executable
      (public_name <project_name>)
      (name main)
      (libraries cmdliner fmt))
```
    Official documentation: https://dune.readthedocs.io/en/stable/tutorials/dune-package-management/dependencies.html

### 4.2. Adding local library modules

In order to include our own library modules in the main code to run in `./bin/`, we just need to add to the `libraries` section of the `./bin/dune` file, like in the section 4.1 Adding third party dependencies above.
