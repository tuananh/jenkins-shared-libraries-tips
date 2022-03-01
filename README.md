# jenkins-shared-libraries

> Tips and tricks working with Jenkins shared libraries

## Jenkins and [kaniko](https://github.com/GoogleContainerTools/kaniko)

Jenkins and kaniko works pretty much out of the box, until you have to use multi-stage Dockerfile, which is quite commonly used these days.

So what's the problem?

Usually, you will want to run a `sh` script to excute kaniko build which looks something like this

```sh
sh """
    /kaniko/executor \
    --context=... \
    --dockerfile=... \
    ...
    --verbosity ...
"""
```

When you do this, Jenkins will use a wrapper script around it. The purpose of this script is to notify Jenkins when the script successes or fails.

The script looks like this

```sh
sh -c ({ while [ -d '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635' -a \! -f '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt' ]; do touch '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-log.txt'; sleep 3; done } & jsc=durable-16856647925e219f4405aa6c51dc26b2; JENKINS_SERVER_COOKIE=$jsc 'sh' -xe  '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/script.sh' > '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-log.txt' 2>&1; echo $? > '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt.tmp'; mv '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt.tmp' '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt'; wait) >&- 2>&- &
```

When you use multi-stage container image build with Kaniko, this script will probably fail, unless you use `--ignore-path` flag. Why is that though?

The script assumes that `mv` command is available which it's not because Kaniko will strip down the paths `/busybox`, making `mv` not available. So the wrapper script will fail even though kaniko build command succeeds. And Jenkins has no way to know your script return status code.

You will then see something like this in Jenkins logs and the advice to increase the timeout which you should not follow.

```
wrapper script does not seem to be touching the log file 
```

The fix here is simple. You just have to use `--ignore-path /busybox` flag and then the wrapper script will work like it supposes too.


## `java.io.NotSerializableException` related errors

