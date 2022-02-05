# GitHub Pages
Create a new branch for github pages:
```bash
#Create a new branch:
git checkout --orphan gh-pages
#We have to make it empty:
git rm -rf .
#Commit & Push the new branch:
git commit --allow-empty -m "root commit"
git push origin gh-pages
#Chech the current branch:
git branch
#Switch to the branch:
git checkout gh-pages
#Switch to master branch:
git checkout master
#Delete branch: When your current branch is main
git branch -d gh-pages
#Delete branch remotely
git push origin --delete gh-pages
```

Lets create an index.html file on our new branch:
```bash
git checkout gh-pages
touch index.html
echo "Hello!" > index.html
git add .
git commit -m "A index html to GitHub page"
git push --set-upstream origin gh-pages
```

And then visit:
```bash
https://rfinland.github.io/ParseChart/index.html
#https://YOURGITHUBUSER.github.io/YOURPROJECT/index.html
#OR just visit:
https://rfinland.github.io/ParseChart/
```
You should able to see what's happen in github actions on your repo
Also you can set theme for your page and add a README.md
For open source projects, GitHub Pages is a great choice to host Helm repositories. 
We’re using the gh-pages branch to store and serve the packaged charts in this part of article. 
After each release we undergo a manual process of packaging and pushing the new chart version to the gh-pages branch.


# Create Helm package
## With cr (chart-releaser)
Install the cr:
```bash
brew tap helm/tap
brew install chart-releaser
```
And then clone the repo:
```bash
git clone https://github.com/rfinland/ParseChart.git
```
Run this command to create package of your helm chart:
```bash
cd ParseChart
helm package charts/Parse --destination .deploy
```
Upload Helm chart packages to GitHub Releases:
```bash
cr upload -o rfinland -r ParseChart -p .deploy -t <YOURTOKEN>
git checkout gh-pages
```
Update a Helm chart repository index.yaml file based on a the given GitHub repository's releases:
```bash
cr index -i ./index.yaml -p .deploy --owner rfinland --charts-repo https://rfinland.github.io/ParseChart --git-repo ParseChart
```
My first package link:
```bash
https://github.com/rfinland/ParseChart/releases/download/Parse-0.1.0/Parse-0.1.0.tgz
```
Push created index.yaml to gh-pages branch:
```bash
git checkout gh-pages
git add index.yaml
git commit -m "Added index yaml"
git push
```
The index.yaml file avaiable on:
```bash
https://rfinland.github.io/ParseChart/index.yaml
```
At last:
```bash
helm repo add Parse https://rfinland.github.io/ParseChart
helm repo update
helm repo list
helm search repo Parse
```
#### Install the package:

##### via The absolute URL: 
```bash
helm install parse https://github.com/rfinland/ParseChart/releases/download/Parse-0.1.0/Parse-0.1.0.tgz
```
##### via The package:
 ```bash
helm install parse parse/parse
```

## With GitHub action
Now I'm going to create the package via GitHub action:
 ```bash
mkdir .github
mkdir .github/workflows
touch .github/workflows/release.yaml 
nano .github/workflows/release.yaml 
 ```
Fill it as below:
 ```bash
name: Release Charts

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"
      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.1.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
		  
 ```
Make a fake release to test the action:
in the ParseChart\charts\Parse\Chart.yaml
I changed:
 ```bash
version: 0.1.0 to version: 0.1.1
 ```
save the change and push our changes
 ```bash
git checkout main
git add .
git commit -m "make an action"
git push
 ```
Visit the action tab  and watch what's going on, As you can see after whole procces of the action done, new release "Parse-0.1.1" published as well.
You can update your helm repo and see the new release.