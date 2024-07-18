---
title: Jupyter Parameterized Kernel Specs
authors: Carlos Herrero, Anastasiia Sliusar
pr-number: 87
date-started: 2022-02-10
shepherd: Johan Mabille
---

# Jupyter Parameterized Kernel Specs

## Problem

When creating a new kernel, we need to define a different "specs" file for each possible parameter combination used by the executable. For example, the xeus-cling kernel supports three different versions of the `C++` language, `C++11`, `C++14`, and `C++17`, which is specified by the `--std` command-line option.

When installing xeus-cling, we install three almost identical kernel specs. The only difference is the last parameter of the execution command `--std=c++X`.

When more than one parameter is available, the number of possible combinations grows in a combinatorics fashion.

Besides, some kernels would benefit from being configurable when launching them. For example, the connection information to the database could be specified through kernel parameters rather than kernel magics.

Sometimes when installing multiple kernels, the JupyterLab launcher panel is crowded with too many options to create a notebook or console with a specific kernel. The kernels that users see in the launcher are just multiple kernel specs for one kernel. We could avoid having that many kernels displayed, adding parameters to the kernel specs, showing only one option per kernel, and offering a modal dialog with the different options the user can choose from when selecting a specific kernel.

## Proposed Enhancement

The solution we are proposing consists of adding parameters to the kernel specs file in the form of a JSON Schema that would be added to the specs metadata. These parameters are then used to populate the `argv` and `env` lists (respectively the command-line arguments and environment variables).

Upon starting a new kernel instance, a front-end form generated from the JSON schema is prompted to the user to fill the parameter values. Many tools are available to generate such forms, such as react-jsonschema-form.

These kernel parameters will be not saved into the notebook due to security reason. The app will have "allow_insecure_kernelspec_params" witch will detect whether we have to accept all kernel spec files as safe or not

## Detailed Explanation

As described in previous sections, we propose to parameterize the kernel specs file. In the example shown below, we can see the kernel specs file from the kernel xeus-cling. We suggest changing the last parameter of the execution command `-std=c++11` to have a variable `-std=${cpp_version}` and adding a new object `parameters` to the metadata of the kernel specs.
 
```=json
{
  "display_name": "C++11",
  "argv": [
      "/home/user/micromamba/envs/kernel_spec/bin/xcpp",
      "-f",
      "{connection_file}",
      "-std=c++11"
  ],
  env: [
    "XEUS_LOGLEVEL=ERROR"
  ],
  "language": "C++11"
}
```
```=json
{
  "display_name": "C++",
  "argv": [
      "/home/user/micromamba/envs/kernel_spec/bin/xcpp",
      "-f",
      "{connection_file}",
      "-std={cpp_version}"
  ],
  env: [
    "XEUS_LOGLEVEL={xeus_log_level}"
  ],
  "language": "C++"
  "metadata": {
    "parameters": {
      "properties": {
        "cpp_version": {
          "type": "string",
          "default": "C++14",
          "enum": ["C++11", "C++14", "C++17"]
        },
        "xeus_log_level": {
          "type": "string",
          "default": "ERROR",
          "enum": ["TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"]
        }
      }
    }
  },
}
```

Note: Using the JSON Schema, we can automate how front-end forms are created directly from the parameters allowing kernels' authors to decide which parameters are necessary and how to validate them. (Note that JupyterLab already makes use of react-jsonschema-form in other parts of its UI).

In the following screenshots, you can see a small demo of how we envision the UI changes in JupyterLab.

  Jupyterlab Launcher      |  Select c++ version
:-------------------------:|:-------------------------:
 ![](./launcher.png)  |  ![](./launcher-select-c-version.png)


![](./notebook-select-kernel.gif)




## Pros and Cons

Pros:

 - A less crowded list of kernels specs

Cons:

 - Changes are required in multiple components of the stack, from the protocol specification to the front-end.
 - Unless we require default values for all parameters, this would be a backward-incompatible change.

## Decisions

- Kernel custom parameters won't be saved into a notebook metadata due to security reason
- The application can be run with `allowed_insecure_kernelspec_params` parameter which allows a user to see a dialog where they can setup custom kernel parameters


## Algorithm
This section describes current implementation.

All kernel spec files are checked whether they are secure when the application is run. If a kernel spec file includes `metadata.parameters`, it means we have the new type of a kernel spec file dedicated for launching a kernel with custom configuration. The new type of kernel spec file can include any structure of JSON schema inside metadata.parameters.

The algorithm looks like the filter of kernel spec files. firstly, it defines whether a kernel spec file is secure or not. There are next secure criteria are used to define whether a kernel spec is secure:

  - if a kernel spec file does not have `metadata.parameters` then it has default behavior that is present now without functionality of parameterized kernels. Such kernel spec file is secure and allowed. In this case when a user clicks on a kernel icon on Launcher then they run a kernel without seeing a dialog for custom kernel parameters. 

  - if all `metadata.parameters` of a kernel spec file do not include free form (text inputs, textarea) where a user can put any information from frontend side. If so such a kernel spec file is save and we show a dialog window for a user.

 If a kernel spec file is not secure then we check whether we setup a flag    `allowed_insecure_kernelspec_params` as true during a running the app. If so, show a dialog window. If not, we still have the bunch of kernel spec files that can be used for customization or not and the task is to run a secure kernel without failing. And if we have the new type of kernel spec file, then we should use its `default values` for each kernel custom parameter. Otherwise the kernel will fail.

 A user can see a dialog which suggests to select kernel parameters if a user clicks on a kernel icon on Launcher or calls a selecting kernel dialog. The `custom_kernel_specs` object inludes user selection and it is proceed on jupyter_client side. If a kernel spec file includes custom kernel variables but a user has not selected anything then `default values` which are in a kernel spec file will be taken. 
 If `custom_kernel_specs`is not empty than the another part of algorithm takes a certain kernel spec file and replace kernel custom variables
