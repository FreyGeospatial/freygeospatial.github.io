---
layout: post
title:  "The Case for Auto-Formatters and Linters: Elevating Code Quality in Software Engineering"
categories: [DevOps, Python, GitHub Actions, bash]
---


I will argue here that standardizing the formatting and styling of code on a team is not immaterial to building and deploying high quality software. Though I give examples using Python and GitHub Actions, the logic is applicable for any other language and CI/CD toolset.

### Ugly Code Reduces Team Efficiency

Imagine a scenario where your colleague was just asked to enhance some code you built. As part of that work, new modules, classes, and functions must be created that will be reused elsewhere.

You have a different coding style than your coworker though, and while reviewing the [diffs](https://en.wikipedia.org/wiki/Diff){:target="_blank"}, you see a pull request marred with conflicting indentation patterns, varied casing styles, and in-line comments describing the functions inplace of docstrings (or worse, no documentation at all). *Hopefully* their code is not approved and merged by another engineer before you have a chance to complete your review.

This scenario is a reality on many engineering teams, and is not ~just~ an annoyance for developers that appreciate pretty code; rather, this reality can cause a multitude of issues including:
1. visual clutter that distracts from the actual code logic being written
1. non-standard style practices that cause IDEs to light up with lint warnings everywhere
1. reduced efficacy of IDE auto-complete functionality when searching for the correct variable, class, function or method name
1. negated functionality of IDEs ability to preview docstring descriptors for modules, classes, functions and methods
1. increased time spent deciphering code that could have been spent building features, fixing bugs, or doing (quite literally) anything else.

Overall, such discrepancies create a hinderance a team's productivity and will only become more problematic as projects grow. Ugly code (and undocumented code- thank goodness for [PEP-257](https://peps.python.org/pep-0257/)) almost certaintly means longer feature-to-production timetables; team-leads at a minimum should encourage adaptation of some guideline to adhere to. Preferably, we should attach linters and autoformatters to our pull requests or CI/CD pipelines.

### ...Enter the Black Formatter
Autoformatters and linters are tools designed to automatically format or check code according to a set of predefined style guidelines or rules; they can also help make code that was written by ten developers and a GPT look like it was written by just one. One of the most popular options for Python is the [Black Formatter](https://black.readthedocs.io/en/stable/){:target="_blank"}, which helps format your Python code according to [PEP-8](https://peps.python.org/pep-0008/){:target="_blank"} industry standards, the most famous of the PEP guidelines.

In the words of its documentation,


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*"By using Black, you agree to cede control over minutiae of hand-formatting. In
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;return, Black gives you speed, determinism, and freedom from pycodestyle
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;nagging about formatting. You will save time and mental energy for more important
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;matters.*


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*Black makes code review faster by producing the smallest diffs possible. 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Blackened code looks the same regardless of the project you’re reading.
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Formatting becomes transparent after a while and you can focus on the content
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;instead."*

If you use VS Code, you can install the Black Formatter as an extension: [https://marketplace.visualstudio.com/items?itemName=ms-python.black-formatter](https://marketplace.visualstudio.com/items?itemName=ms-python.black-formatter){:target="_blank"}. 

You can also install black formatter as a CLI utility: `pip install black`.

### Other Useful Linters and Extensions:
You can use other linters as either an alternative or in addition to Black. Some recommended ones include:
- flake8, flakeheaven, flake-8-docstrings, flake8-docstrings-complete
- pylint
- ruff

Each has their own advantages.

### Linter Usage and Enforcement:

There are many strategies for how to best enforce linting and formatting on software engineering teams. I will suggest an approach that works best for me, and show how this can be done using CI/CD.


#### Requirements:

To follow this tutorial in its entirety, you must:
1. Have VS Code installed
1. Have Python installed, and a basic understanding of python syntax
1. Install these VS Code Extensions:
    - Black formatter extension
    - Python extension
1. Have a GitHub account and public repository to hold examples
1. A basic understanding of `git` concepts, CI/CD, and preferably experience using GitHub Actions or Azure Pipelines

It will also be helpful- but not explicitly required- to use these tools locally as you work through this tutorial:
- `black` formatter CLI utility: [https://github.com/psf/black](https://github.com/psf/black)
- `act` utility for running github actions locally: [https://github.com/nektos/act](https://github.com/nektos/act)


#### Workflow:
The easiest methods I have seen has been to first set up your IDE to autoformat your python code automatically upon saving.

After installing the Black Formatter VS Code Extension, I go into my VS Code user settings. On MacOS, these can usually be found at: `/Users/<user>/Library/Application Support/Code/User/settings.json`.

Add this to the json:

```json
    "editor.formatOnSave": false,
    "[python]": {
        "editor.formatOnType": true,
        "editor.defaultFormatter": "ms-python.black-formatter",
        "editor.formatOnSave": true,
    },
    "black-formatter.showNotifications": "always"
```

With these settings, saving a python file will automatically be autoformatted according to Black style guides. If you keep `formatOnSave` set to false, you could alternatively right-click the editor, and select "format document" for the formatter to run.

On a team though, it can be hard to enforce contributor IDE settings. A CI/CD rule to check that code is styled according to team guidelines is a necessary second-check.

To check ONLY those files that have been modified against the `main` branch, first add this shell script to your .github folder under a "scripts" subdirectory:

```bash
# echos modified python files, but excludes those deleted
changed_files=$(git diff --diff-filter=d --name-only $(git merge-base HEAD remotes/origin/main) HEAD | grep .py)
echo ${changed_files}
```

Then, add these steps to your Actions workflow:

```yaml
      - name: install black formatter
        run: pip install black

      - name: Save modified files to environmental variables
        id: get_modified_files
        run: echo MODIFIED_FILES=$(./.github/scripts/modified_files.sh) >> $GITHUB_ENV
      
      - name: return modified python files
        run: echo $(./.github/scripts/modified_files.sh)

      - name: Lint modified Python files with Black
        run: |
          if [ -z "$MODIFIED_FILES" ]; then
            echo "No Python files modified."
          else
            black --check --diff --color $MODIFIED_FILES
          fi
```

If there was any improperly formatted Python code, the deployment would fail. For the full code, you visit my Github repository: https://github.com/FreyGeospatial/github_actions. And feel free to fork my repo as needed!

Overall, this is a pretty simple example of what you can do to enforce a cleaner codebase on your team. I encourage you to explore other linters, including `Flake8`, or even `ruff`, the latter of which has gained popularity recently for being very computationally performant. 

I hope you have found this helpful. If you have any comments, questions, or feedback- please reach out! And as always, happy coding!
