# SageMaker: SSH to notebook instances

If you’re using SageMaker as a development machine, you’ll need SSH access to notebook instances sooner or later.

You’ll want to copy your notebook over with scp.
Or you’ll get stuck and want to connect a remote debugger with PyCharm or VsCode.
Or just access your GitHub repositories without leaving your private key lying around, accessible to anybody who has access to AWS console in your company.

You will Google with no luck. Then you’ll ask AWS and figure out it’s not supported.

Luckily, it’s possible to make it work. I use it every day and it’s super useful.

Like other SageMaker guides here, this one will show you how to set up SSH access to SageMaker notebook instances in just a few minutes.

# Why isn’t Sagemaker SSH natively supported by AWS?

SageMaker notebook instances are managed by AWS. They are EC2 instances and part of your account, but if you list all EC2 machines, you won’t see them.

That gives you less control over SageMaker instances. You can control basic network settings like VPC, security groups, and whether it will have direct internet access or not, but you can’t assign a public IP address to it.

Because SSH needs some way to talk to the notebook instance and you can’t assign a public IP address, your machine has no way to talk to the notebook instance directly.

## But I can access Jupyter from my browser?!

Correct, you can. The way your browser talks to the notebook instance is through proxy run by AWS. It does not talk directly to the notebook instance.

We’ll do something similar to enable SSH access.

# How does the setup work?

As you’ve probably figured it out already, we just have to drill some way for your machine and notebook instance to talk to each other.

When two machines can’t talk to each other, but they both know a third machine, they can ask that third machine to exchange their messages. That’s exactly what we’ll do.

I’ll show you two solutions. One using ngrok reverse proxy (3rd party proxy) and the other one proxying through other EC2 instance (usually called bastion) in your account. If you’re not sure which one you should use, one with ngrok is probably the right one for you.

# SageMaker SSH

## Prepare lifecycle configuration

Before we start with any specific setup, we have to make sure there is a lifecycle configuration attached to your notebook instance.

If there is no configuration, we will create a new one with the name “better-sagemaker” and configure the ssh in it.

### Steps:

1. Login to AWS console
2. Stop your notebook instance (and wait for the instance to stop)
3. Make sure you have [jq tool installed](https://stedolan.github.io/jq/download/)
Copy the following script into your terminal (on your computer, not SageMaker).

Do it step by step.

If you know there is one and you know the name, you can just fill INSTANCE_NAME and CONFIGURATION_NAME variables and skip the rest.

```
# fill in the instance name here
INSTANCE_NAME="team-ml-mario"
```

```
CONFIGURATION_NAME=$(aws sagemaker describe-notebook-instance --notebook-instance-name "${INSTANCE_NAME}" | jq -e '.NotebookInstanceLifecycleConfigName | select (.!=null)' | tr -d '"')
echo "Configuration \"$CONFIGURATION_NAME\" attached to notebook instance $INSTANCE_NAME"
```
```
if [[ -z "$CONFIGURATION_NAME" ]]; then
    # there is no attached configuration name, create a new one
    CONFIGURATION_NAME="better-sagemaker"
    echo "Creating new configuration $CONFIGURATION_NAME..."
    aws sagemaker create-notebook-instance-lifecycle-config \
        --notebook-instance-lifecycle-config-name "$CONFIGURATION_NAME" \
        --on-start Content=$(echo '#!/usr/bin/env bash'| base64) \
        --on-create Content=$(echo '#!/usr/bin/env bash' | base64)

    # attaching lifecycle configuration to the notebook instance
    echo "Attaching configuration $CONFIGURATION_NAME to ${INSTANCE_NAME}..."
    aws sagemaker update-notebook-instance \
        --notebook-instance-name "$INSTANCE_NAME" \
        --lifecycle-config-name "$CONFIGURATION_NAME"
fi
```
# Set up SSH with Ngrok

Ngrok is a third party reverse proxy that will tunnel our your traffic. Don’t worry, SSH is encrypted end to end so ngrok can’t read any of it.

To get permissions for creating TCP tunnels, you’ll need a [free ngrok account](https://dashboard.ngrok.com/get-started/setup) and find an authentication token in their dashboard.


Once you grab it, fill in the token below and run it step by step.

```
export NGROK_AUTH_TOKEN="FILL_TOKEN_HERE"
```

```
echo "Downloading on-start.sh..."
# save the existing on-start script into on-start.sh
aws sagemaker describe-notebook-instance-lifecycle-config --notebook-instance-lifecycle-config-name "$CONFIGURATION_NAME" | jq '.OnStart[0].Content'  | tr -d '"' | base64 --decode > on-start.sh

echo "Adding SSH setup to on-start.sh..."
# add the code to persist conda environments
echo '' >> on-start.sh
echo '# set up ngrok ssh tunnel' >> on-start.sh
echo "export NGROK_AUTH_TOKEN=\"${NGROK_AUTH_TOKEN}\"" >> on-start.sh
echo 'curl https://raw.githubusercontent.com/mariokostelac/sagemaker-setup/master/scripts/ssh/on-start-ngrok.sh | bash' >> on-start.sh

echo "Uploading on-start.sh..."
# update the lifecycle configuration config with updated on-start.sh script
aws sagemaker update-notebook-instance-lifecycle-config \
    --notebook-instance-lifecycle-config-name "$CONFIGURATION_NAME" \
    --on-start Content="$((cat on-start.sh)| base64)"
```

That just made sure that ngrok starts every time the notebook instance starts, with your authentication token.

The ngrok config is located in /home/ec2-user/SageMaker/.ngrok/config.yml. If you want to add a fixed remote_addr to have the stable address to connect to (you’ll need a pro account for this), you can change it in the config file.

Also, if the tunnel ever closes (connections can drop), you can start the tunnel again by running start-ssh-ngrok in your SageMaker terminal. It will run in the background.

And that’s almost all the setup we need. You can start your instance now.

We just need to set up your ssh keys. 

# Set up SSH keys
So far we’ve set up the way to talk to your SageMaker instance, but we still haven’t defined who exactly can access these machines.

To do so, we’ll have to add public keys for all users that need access to ~/SageMaker/ssh/authorized_keys.

That likely means that you will have to copy whatever is in your local ~/.ssh/id_rsa.pub into ~/SageMaker/ssh/authorized_keys on SageMaker instance.

Once you’ve done that, make sure you run copy-ssh-keys on SageMaker terminal so these keys are copied to the correct location.

# SSHing into your notebook instance
Every time your machine starts, the address you have to connect to will change. It’s unfortunate and can be fixed by more scripting, but I won’t go into details in this post (feel free to contact me if you really want it).

Every time it changes, it will be written to ~/SageMaker/SSH_INSTRUCTIONS file. If you click on Open Jupyter in AWS Console, it should be in the directory shown by default.

That’s it, just follow these instructions. For me, it was ssh -p 17887 ec2-user@0.tcp.ngrok.io and I was in!
