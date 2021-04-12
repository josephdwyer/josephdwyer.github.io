---
layout: post
title: Block GlobalSection(Performance) from .sln
published: false
---

Visual Studio 2015 (in my opinion) is a great improvement over previous iterations, with a major exception:  

### Transient and user-specific information stored in .sln

Using the built-in profiling tools adds a section to the solution.  

```
GlobalSection(Performance) = preSolution
    HasPerformanceSessions = true
EndGlobalSection
```

These spurious changes cause havoc in source control systems.
Code reviews include meaningless changes and merge conflicts are likely.
This behavior [was an oversight by the Visual Studio team](https://connect.microsoft.com/VisualStudio/feedback/details/1951562).
`.sln` files should be included in source control and shared by team members.

> A solution is a structure for organizing projects in Visual Studio. The solution maintains the state information for projects in .sln (text-based, shared) and .suo (binary, user-specific solution options) files. For further information on .suo files, see Solution User Options (.Suo) File.
>
> -- [Solution (.Sln) File MSDN](https://msdn.microsoft.com/en-us/library/bb165951.aspx)

The [solution to this problem](http://stackoverflow.com/questions/14981323/how-to-force-visual-studio-not-to-add-globalsectionperformance-section) is to *just* delete the generated `.vsp` and `.psess` files.
__This is inadequate,__ it is easy to include the changes causing headaches for your team members.

### A better way

At least if your source control supports pre-commit hooks (a-la git).
I use Git Bash, your mileage may vary depending on what tools you use.

Getting your team's environments configured can be a hassle and each new hire will have to repeat the process.
So I recommend storing the scripts in your project's source control and automating the setup process.

Create a subdirectory in your repository that your application will ignore something like `dev_setup/`.

First, we need to create a script that will remove the `GlobalSection(Performance)`.
There are many ways to do this, but one way is to use a multi-line regular expression.
 Create this file in your subdirectory:

##### dev_setup/remove_perf_section.ps1
```
<#
.DESCRIPTION
Removes performance section from file.

  GlobalSection(Performance) = preSolution
      HasPerformanceSessions = true
  EndGlobalSection

.PARAMETER $slnFile
Local path to the .sln to modify
.EXAMPLE
remove_perf_section.ps1 .\exe\MyProject.sln
#>
param($slnFile)

$oldCode = [Regex]::Escape("`tGlobalSection(Performance) = preSolution`r`n`t`tHasPerformanceSessions = true`r`n`tEndGlobalSection`r`n")

$fileContent = Get-Content $slnFile -Raw
$newFileContent = $fileContent -replace $oldCode, ""
Set-Content -Path $slnFile -Value $newFileContent
```

Next, we need to call our script if `GlobalSection(Performance)` is being committed. Replace `subdir/MyProject.sln` with the path to your solution file.

##### dev_setup/hooks/pre-commit
```
#!/usr/bin/bash

# Visual Studio adds a performance section to the .sln
# when you use the profiler. This is super annoying and should not be in the .sln, this script removes it.

slnFile="subdir/MyProject.sln"

git diff --cached --name-only --diff-filter=ACMR $slnFile | while read filename; do
	echo "Replacing Performance Section"
	PowerShell.exe -NoProfile -NonInteractive -ExecutionPolicy Unrestricted -Command "dev_setup\remove_perf_section.ps1" $slnFile
	git add $slnFile
done \
|| exit $?
```

Next, add some setup functions:

##### dev_setup/setup.bash
```
#!/usr/bin/bash

# copy git hooks into directory
install_git_hooks() {
  cp++ dev_setup/hooks/pre-commit .git/hooks/pre-commit
}

# cp and create intermediate directories
cp++() { mkdir -p `dirname $2` && cp "$1" "$2"; }
```

Finally, add instructions for your teammates to your readme or other documentation:

    To setup git hooks:
    ```
    source dev_setup/setup.bash
    install_git_hooks
    ```

Test it out the `GlobalSection(Performance)` should not make it into your commits.
