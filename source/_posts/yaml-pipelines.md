---
title: YAML pipelines in Azure DevOps
date: 2020-03-29 18:00:00
tags: 
- Azure DevOps
- YAML
- pipeline
---

In this blog post, we are going to talk about YAML pipelines in Azure DevOps. Microsoft has been pushing YAML as the new default way of setting up build/release pipelines for some time now. When it was first set up, documentation was lacking and not all tasks that were available in the visual designer were available in YAML. Thus, it makes sense that the developers who looked into it at that moment were put off. However, much has changed since then and YAML pipelines have grown into a mature framework of setting up your build/release pipelines. Let's take a look.

## What is YAML?
> **YAML** is a human friendly data serialization standard for all programming languages.

The YAML language is very clean and, like Python, used indentation to indicate nesting. Like JSON, it uses `[]` for lists and `{}` for maps. It is often used as a language to define settings or configuration files. For an overview of the YAML language, take a look at the [YAML cheat sheet](https://yaml.org/refcard.html).

## Why would you use YAML pipelines?
Moving to have *everything as code* is making is in an uptrend, for good reasons. Having your build/deployment pipeline as code, in version control, gives you some powerful benefits over having it defined in the visual designer. Sure, you lose the purely visual representation as you *are* working in a text file, but you gain the following:

+ your pipeline can be validated through code reviews by your colleagues in pull-requests or in some other way
+ if a change is introduced that *does* break the pipeline, restoring it is as easy as reverting the commit
+ should a pipeline have to be set up again, it can be done by selecting the YAML definition and you're done
+ you can have separate YAML pipeline definitions in your code, per project, per solution, multiple per project, you decide
+ you can set up different templates, like tasks, and combine them, like in task groups, and re-use them very easily
+ environments and (conditional) multi-stage deployments are also defined in the YAML pipeline definition

In the end, you can get to a situation where you don't need to look at the pipeline itself anymore to deploy your code, even with pipeline changes:
1. edit your code
2. edit the yaml file
3. push to the repo
4. Azure Pipelines runs
5. deployed to target

## How are Azure pipelines set up?
Azure pipelines contain the following key concepts:
+ *triggers* tell pipelines to run;
+ *stages* are used to organize jobs, can require approvals;
+ *environments* are the targets for deployment and can define their own deployment strategies and resources
+ *jobs* are used to organize steps, run on an agent or can run agentless
+ *agents* are installable software running a job,
+ *steps* are tasks or scripts, the smallest item of a pipeline
+ *scripts* are custom code run
+ *tasks* are pre-defined building blocks defined by Azure DevOps

More on the key concepts of Azure Pipelines can be found [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/key-pipelines-concepts)

You don't have to use all layers of this kind of hierarchy, the simplest pipeline definitions can consist of just a single step, where stages and jobs are implicit.

It is possible to set a filter on branch or path for the trigger, to only run after a commit or code change in a certain folder respectively. It is possible to import other YAML templates to re-use certain components over multiple pipelines.

YAML pipelines can be set up to get variables from Azure Keyvault to use as input parameters for the pipeline. It is also possible to override input parameters from Azure Pipelines manually i.e. for testing purposes.

## How to start with YAML pipelines
So, after all this, how does one start using YAML pipelines?

+ Start with one of the basic starter files on Azure DevOps, where after selecting your project type, you will get a starting point and a url to find more information for your specific project type
+ Start with one of the templates on [github](https://github.com/microsoft/azure-pipelines-yaml)
+ You can start with an empty file and use the [documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/) to guide you (not recommended).


In the Azure Pipelines online editor, you can find tasks similar to the ones in the visual designer where you can enter the required parameters and add the code to the YAML pipeline. This also enables you to click on the `settings` tag that appears on top of default tasks to go back to the input fields for that step.

And in the end, if all else fails and you cannot find the task you are looking for in the online YAML editor... You can always go to the visual designer, create your step there, export it as YAML, copy it in your YAML pipeline and still enjoy all the benefits your pipeline in code has to offer!