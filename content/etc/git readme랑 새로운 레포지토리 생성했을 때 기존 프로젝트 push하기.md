### Step 1: Open Your Terminal

Open your terminal and navigate to the root directory of your Spring application (the folder that contains your `build.gradle` and `src` folder).

### Step 2: Initialize Git (If you haven't already)

If your Spring project isn't a Git repository yet, initialize it.

Bash

```
git init
```

### Step 3: Link the Remote Repository

Connect your local folder to your GitHub/GitLab repository. Replace the URL with your actual repository URL.

Bash

```
git remote add origin https://github.com/your-username/your-repo-name.git
```

### Step 4: Fetch the Remote Branches

This tells your local Git to download the data about the `main` branch and the new branch you created, but it won't touch your local files yet.

Bash

```
git fetch
```

### Step 5: Switch to Your New Branch

Now, switch to the branch you created on the remote. Because you haven't committed anything locally yet, Git will safely download the `README.md` into your folder without deleting your Spring files.

Bash

```
git switch <your-new-branch-name>
```

_(Note: If Git throws an error saying a local file would be overwritten, it is usually because Spring Initializr created a `README.md` or `HELP.md` locally that conflicts with the remote one. Simply delete or rename your local `README.md`, and run the command again)._

### Step 6: Stage and Commit Your Code

Before running this, make sure your project has a `.gitignore` file (Spring Initializr generates one by default) so you don't accidentally push your compiled `build/` directory or `.idea/` IDE settings.

Stage all your Spring files and commit them:

Bash

```
git add .
git commit -m "Add existing Spring Boot application code"
```

### Step 7: Push to the Remote Repository

Finally, push your code up to the remote branch. The `-u` flag sets the "upstream" tracking, so next time you can just type `git push`.

Bash

```
git push -u origin <your-new-branch-name>
```

Your Spring application will now be safely stored on the new branch, right alongside the `README.md`!