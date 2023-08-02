## RHTAP Templates

This project contains the code source needed to demo buildpack on RHTAP such as:
- Quarkus REST service axposing a Hello endpoint `/hello` and `/hello/greeting/{name}` 
- Tekton PipelineRun able to build a container image using buildpack.

## HowTo

To create a new GitHub repository and import the needed files, perform the following actions:

- Git auth
`gh auth login --with-token <YOUR_GITHUB_TOKEN>`

- Create a GitHub repository

```bash
ORG_NAME=halkyonio
REPO_TEMPLATE=rhtap-templates
REPO_DEMO_NAME=rhtap-buildpack-demo-1
REPO_DEMO_TITLE="RHTAP Buildpack Demo 1"

gh repo delete $ORG_NAME/$REPO_DEMO_NAME --yes
gh repo create $ORG_NAME/$REPO_DEMO_NAME --public

rm -rf $REPO_DEMO_NAME; mkdir $REPO_DEMO_NAME; cd $REPO_DEMO_NAME
git init
echo "## RHTAP Buildpack Demo 1" > README.md
git add .

## Import runtime code
BRANCH=main
wget https://github.com/$ORG_NAME/$REPO_TEMPLATE/archive/$BRANCH.zip
unzip $BRANCH.zip
mv $REPO_TEMPLATE-$BRANCH/quarkus-hello/.mvn/ .
mv $REPO_TEMPLATE-$BRANCH/quarkus-hello/* .

echo "$REPO_TEMPLATE-$BRANCH/" > .gitignore
echo "$BRANCH.zip" >> .gitignore
echo "target/" >> .gitignore

git add .
git commit -m "Upload quarkus hello runtime"

git branch -M main
git remote add origin git@github.com:$ORG_NAME/$REPO_DEMO_NAME.git
git push -u origin main
cd ..
```
- Test it locally
```bash
cd $REPO_DEMO_NAME
mvn clean compile; mvn quarkus:dev

http :8080/hello
http :8080/hello/greeting/charles
cd ..
```
- Create a `.Tekton` folder
```bash
cd $REPO_DEMO_NAME
mkdir .tekton
mv $REPO_TEMPLATE-$BRANCH/tekton/template-push.yaml .tekton/$REPO_DEMO_NAME-push.yaml
git add .tekton/$REPO_DEMO_NAME-push.yaml
```

- Customize the RHTAP PipelineRun
```bash
APPLICATION_NAME=$REPO_DEMO_NAME
COMPONENT_NAME="quarkus-hello"
PAC_NAME=$COMPONENT_NAME-$(openssl rand -base64 4 | tr -dc 'a-zA-Z0-9' | cut -c1-5)
PAC_YAML_FILE=".tekton/$REPO_DEMO_NAME-push.yaml"
# REGISTRY_URL=quay.io/redhat-user-workloads/cmoullia-tenant/$REPO_DEMO_NAME
REGISTRY_URL=quay.io/$ORG_NAME/$REPO_DEMO_NAME

sed -i.bak "s/#APPLICATION_NAME#/$APPLICATION_NAME/g" $PAC_YAML_FILE
sed -i.bak "s/#COMPONENT_NAME#/"$COMPONENT_NAME"/g" $PAC_YAML_FILE
sed -i.bak "s/#PAC_NAME#/"$PAC_NAME"/g" $PAC_YAML_FILE
sed -i.bak "s/#REGISTRY_URL#/"$REGISTRY_URL"/g"  $PAC_YAML_FILE
rm $PAC_YAML_FILE.bak
git commit -sm "Add the tekton push file" .tekton/$REPO_DEMO_NAME-push.yaml; git push
cd ..
```
- Import it as documented here: https://redhat-appstudio.github.io/docs.appstudio.io/Documentation/main/how-to-guides/Import-code/proc_importing_code/

- Cleaning
```bash
rm $BRANCH.zip; rm -r $REPO_TEMPLATE-$BRANCH
```



## Issue

### Full image path not supported

The lifecycle component and most probably google container library (used by lifecycle to access the registry) do not support such advanced feature: https://kubernetes.io/docs/concepts/containers/images/#kubelet-credential-provider
The consquence is that if several secrets are attached to the `appstudio-pipeline` service account and subsequently by the pod running lifecycle, then
lifecycle, at the analysis step, will raise an issue if it doesn't get as first entry of the `auths:` config file (from mounted secrets) the full image path matching the image name declared
as output image.