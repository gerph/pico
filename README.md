# Pico (and Pilot)

## Introduction

Pico is a simple editor for use with a text display. Pilot is a file
browser for use with a text display.

This is a port of the Pico editor to RISC OS, which was created around
1997, and updated a few times since then. This is a build from the Pine
4.63 sources, which were released around 2005.

## Usage

The application !Pico will open a Taskwindow with the tool open.
The command 'pico' will edit files from the command line.

Pilot doesn't work very well, and hasn't been given a whole lot of
testing.

## Continuous Integration (CI) support

This release is intended to demonstrate the continuous integration environment
provided by the JFPatch-as-a-service system. Two mechanisms are provided for
building the module - through GitLab CI, and through GitHub's workflows.

Documentation for the `.robuild.yaml` file format can be found on the
[JFPatch-as-a-service website](https://jfpatch.riscos.online/robuildyaml.html),
and the example here just invokes the AMU tool to perform the build.

* For GitHub workflows, see the file `.github/workflows/jfpatch.yml`.
* For GitLab CI, see the file `.gitlab-ci.yml`.
