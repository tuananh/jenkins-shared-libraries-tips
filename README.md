# Jenkins Share Libraries Tips

> Tips and tricks with Jenkins shared libraries

If you still have to work with Jenkins in 2022, you may find this useful to you.


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

When you do this, Jenkins will use a wrapper script around it. The purpose of this script is to notify Jenkins when the `sh` script succeeds or fails.

The wrapper script looks like this

```sh
sh -c ({ while [ -d '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635' -a \! -f '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt' ]; do touch '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-log.txt'; sleep 3; done } & jsc=durable-16856647925e219f4405aa6c51dc26b2; JENKINS_SERVER_COOKIE=$jsc 'sh' -xe  '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/script.sh' > '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-log.txt' 2>&1; echo $? > '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt.tmp'; mv '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt.tmp' '/home/jenkins/agent/workspace/test@tmp/durable-ca5ae635/jenkins-result.txt'; wait) >&- 2>&- &
```

When you use multi-stage container image build with Kaniko, this script will probably fail, unless you use `--ignore-path` flag. Why is that though?

The script assumes that `mv` command is available which it's not because Kaniko will strip down the paths `/busybox`, making `mv` not available. So the wrapper script will fail even though kaniko build command succeeds. And Jenkins has no way to know your script return status code.

You will then see something like this in Jenkins logs and the advice to increase the timeout which you should not follow.

```
wrapper script does not seem to be touching the log file 
```

The fix here is simple. You just have to use `--ignore-path=/busybox` flag and then the wrapper script will work like it supposes too.

Refs:

- https://github.com/GoogleContainerTools/kaniko/pull/1622
- https://github.com/GoogleContainerTools/kaniko/issues/1275
- https://github.com/GoogleContainerTools/kaniko/issues/1449


## `java.io.NotSerializableException` related errors

> *Pipeline restricts all variables to Serializable types, so keeping Pipeline logic simple helps avoid a NotSerializableException - see appendix at the bottom.*
> - [Best practices for scalable pipeline code - Jenkins's blog](https://www.jenkins.io/blog/2017/02/01/pipeline-scalability-best-practice/)

This is another pain point of Jenkins. Avoid using non-serializable types (only safe in @NonCPS functions). Example of non-serializable types are iterators, regex matchers, etc...

Another example that I faced with is with the `groovy.text.SimpleTemplateEngine`. While `SimpleTemplateEngine` is not serializable, `groovy.text.StreamingTemplateEngine` is. So you may want to switch over to it.

