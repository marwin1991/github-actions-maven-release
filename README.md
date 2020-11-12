# github action maven release

The GitHub Action for Maven releases wraps the Maven CLI to enable Maven release.
For example, you can use this action for auto-incrementing your project version and release your java artifacts.

This github action is bot friendly: You can configure the credentials of a bot user, which would be used during the incremental commit.
The commits by the bot can also be signed, giving you the guaranty that only the bot can release in your repo. Additionally,
this give you a clean git history by highlighting nicely which commits where resulting from your CI.

## Features

Obviously, this github actions uses maven release plugin. Although, we did add on top a few features that you may like.

Maven release uses Git behind it, therefore there were a few features related in customising the git configuration:
- Signing the commits (GPG) resulting from the maven release [[GPG](#setup-a-gpg-key)]
- Authenticating to private repository using an SSH key [[SSH](#setup-with-ssh)]
- Configuring the git username and email [[Bot](#customise-the-bot-name)]

You may want to configure a bit maven too. We added the following features:
- Specify the maven project path. In other words, if your maven project is not at the root of your repo, you can configure a sub path. [[Custom project path](#customise-the-m2-folder-path)] 
- Configure a private maven repository [[Private maven repo](#setup-a-private-maven-repository)]
- Setup custom maven arguments and/or options to be used when calling maven commands [[Maven options](#maven-options)]
- Configure a custom M2 folder [[Custom M2](#customise-the-m2-folder-path)]
- Print the timestamp for every maven logs. Handy for troubleshooting performance issues in your CI. [[Log timestamp](#log-timestamp)]

For the maven releases, we got also some dedicated functionalities:
- Skip the maven perform [[Skip perform](#skipping-perform)]
- Roll back the maven perform if it failed to perform the release
- Increment the major or minor version (by default, it's the path version that is increased) [[Major Minor version](#increase-major-or-minor-version)]


# Usage

## Setup your pom.xml for maven release

Before you even begin setting up this github action, you would need to set up your pom.xml first to be ready for maven releases.
We recommend you to refer to the maven release plugin documentation for more details: https://maven.apache.org/maven-release/maven-release-plugin/

Nevertheless, we will give you some essential setups

### Configure the SCM

You got two choices here:
- Using SSH URL (Recommended)
```xml
    <scm>
        <connection>scm:git:${project.scm.url}</connection>
        <developerConnection>scm:git:${project.scm.url}</developerConnection>
        <url>git@github.com:idhub-io/idhub-api.git</url>
        <tag>HEAD</tag>
    </scm>
```
- Using HTTPS URL
```xml
	<scm>
        <connection>scm:git:${project.scm.url}</connection>
        <developerConnection>scm:git:${project.scm.url}</developerConnection>
		<url>https://github.com/YOUR_REPO.git</url>
		<tag>HEAD</tag>
	</scm>
```

In the case of SSH, it will use the `ssh-private-key`  to authenticate with the upstream.
In the case of HTTPS, maven releases will use the `access-token` in this github actions to authenticate with the upstream.

Note: SSH is more elegant and usually the easiest one to setup due to the large amount of documents online on this subject.

### maven release plugin

Add the maven release plugin dependency to your project

```xml
    <plugin>
        <artifactId>maven-release-plugin</artifactId>
        <version>XXX</version>
        <configuration>
            <scmCommentPrefix>[ci skip]</scmCommentPrefix>
        </configuration>
    </plugin>
```

Personally, I usually the prefix `[ci skip]` which allows me to skip more easily any commits generated by the bot from the CI.

## Setup the maven release github actions

### Basic setup
For a simple repo with not much protection and private dependency, you can do:

```
 - name: Release
      uses: qcastel/github-actions-maven-release@master
      with:
        access-token: ${{ secrets.GITHUB_ACCESS_TOKEN }}
```

### Setup with SSH

Although you may found better to use a SSH key instead. For this, generate an SSH key with the method of your choice, or use an existing one.
Personally, I like generating an SSH inside a temporary docker image and configure it as a deploy key in my repository:

```
docker run -it qcastel/maven-release:latest  bash
```

```
ssh-keygen -b 2048 -t rsa -f /tmp/sshkey -q -N ""
export SSH_PRIVATE_KEY=$(base64 /tmp/sshkey)
export SSH_PUBLIC_KEY=$(base64 /tmp/sshkey.pub)
echo -n "Copy the following SSH private key and add it to your repo secrets under the name 'SSH_PRIVATE_KEY':"
echo $SSH_PRIVATE_KEY
echo "Copy the encoded SSH public key and add it as one of your repo deploy keys with write access:"
echo $SSH_PUBLIC_KEY

exit 
```

Copy `SSH_PRIVATE_KEY` and add it as a new secret.
![img/Add-ssh-secrets.png](img/Add-ssh-secrets.png?raw=true)

Copy `SSH_PUBLIC_KEY` and add it as a new deployment key with write access.
![img/add-deploy-key.png](img/add-deploy-key.png?raw=true)

Finally, setup the github action with:

```yaml
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

### log Timestamp

It can be quite difficult to troubleshoot any performance issue on your CI, due to the lack of timestamp from maven by default.
An example of it particular handy, is when you private maven repository is having performance issue that is affecting your CI.

We added the timestamp by default, you don't need to do anything particular to enable this feature.

The logs should look like:

```
14:27:09,491 [INFO] Downloading from spring-snapshots: https://repo.spring.io/snapshot/io/projectreactor/reactor-bom/Dysprosium-SR13/reactor-bom-Dysprosium-SR13.pom
``` 
 
### Maven options

#### Adding maven arguments

You can add some maven arguments, which is handy for skipping tests:

```yaml
        with:
            maven-args: "-Dmaven.javadoc.skip=true -DskipTests -DskipITs -Ddockerfile.skip -DdockerCompose.skip"
```

#### Adding maven options

You can add some maven options. At the difference of the maven arguments, those one are explicitly for the maven release plugin.
See https://maven.apache.org/maven-release/maven-release-plugin/prepare-mojo.html.

```yaml
        with:
            maven-options: "-DbranchName=hotfix"
```

### Customise the bot name

You can simply customise the bot name as follows:


```yaml
        with:
            git-release-bot-name: "release-bot"
            git-release-bot-email: "release-bot@example.com"
```

### Customise the default branch
You may not release from your master branch. You can customise the branch name, here `release`, as follows:

```yaml
        with:
            release-branch-name: "release"

```

### Skipping perform
If for a reason, you need to skip the maven release perfom, you can disable it as follow:

```yaml
        with:
            skip-perform: false
```


### Increase major or minor version
By default, maven release will increase the path version. If you are interested to actually increment the major or minor version, you can use
the following options:

#### For major version increment

_1.0.0-SNAPSHOT -> 2.0.0-SNAPSHOT_

```yaml
        with:
            version-major: true
```

#### For minor version increment

_1.0.0-SNAPSHOT -> 1.2.0-SNAPSHOT_

```yaml
        with:
            version-minor: true
```

#### Customise the M2 folder path

It's quite common for setting up a caching of your dependencies, that you will be interested to customise the .m2 localisation folder.

```yaml
        with:
            m2-home-folder: '/your-custom-path/.m2'
```


### Setup a GPG key
If you want to set up a GPG key, you can do it by injecting your key via the secrets:

Note: `GITHUB_GPG_KEY` needs to be base64 encoded.
if you haven't setup a GPG key yet, see next section.

```yaml
      with:
        gpg-enabled: "true"
        gpg-key-id: ${{ secrets.GITHUB_GPG_KEY_ID }}
        gpg-key: ${{ secrets.GITHUB_GPG_KEY }}
```


### Setup a private maven repository

If you got a private maven repo to set up in the settings.xml, you can do:
Note: we recommend putting those values in your repo secrets.

```yaml
      with:
        maven-repo-server-id: ${{ secrets.MVN_REPO_PRIVATE_REPO_USER }}
        maven-repo-server-username: ${{ secrets.MVN_REPO_PRIVATE_REPO_USER }}
        maven-repo-server-password: ${{ secrets.MVN_REPO_PRIVATE_REPO_PASSWORD }}
```


### Configure your maven project

You may also be in the case where you got more than one maven projects inside the repo. We added an option that will make the release job move to the according directly before running the release:

```
    with:
        maven-project-folder: "sub-folder/"
```


## Setup the bot gpg key

Setting up a gpg key for your bot is a good security feature. This way, you can enforce sign commits in your repo,
even for your release bot.

![Screenshot-2019-11-28-at-20-47-06.png](https://i.postimg.cc/9F6cxpqm/Screenshot-2019-11-28-at-20-47-06.png)

- Create dedicate github account for your bot and add him into your team for your git repo.
- Create a new GPG key: https://help.github.com/en/github/authenticating-to-github/generating-a-new-gpg-key

This github action needs the key ID and the key base64 encoded.

```$xslt
        gpg-key-id: ${{ secrets.GITHUB_GPG_KEY_ID }}
        gpg-key: ${{ secrets.GITHUB_GPG_KEY }}
```

### Get the KID

You can get the key ID doing the following:

```$xslt
gpg --list-secret-keys --keyid-format LONG

sec   rsa2048/3EFC3104C0088B08 2019-11-28 [SC]
      CBFD9020DAC388A77C68385C3EFC3104C0088B08
uid                 [ultimate] bot-openbanking4-dev (it's the bot openbanking4.dev key) <bot@openbanking4.dev>
ssb   rsa2048/7D1523C9952204C1 2019-11-28 [E]

```
The key ID for my bot is 3EFC3104C0088B08. Add this value into your github secret for this repo, under `GITHUB_GPG_KEY_ID`
PS: the key id is not really a secret but we found more elegant to store it there than in plain text in the github action yml

### Get the GPG private key

Now we need the raw key and base64 encode
```$xslt
gpg --export-secret-keys --armor 3EFC3104C0088B08 | base64
```

Copy the result and add it in your github repo secrets under `GITHUB_GPG_KEY`.

Go the bot account in github and import this GPG key into its profile.


# License
The Dockerfile and associated scripts and documentation in this project are release under the MIT License.

