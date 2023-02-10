---
published: false
title: 'Deploying a Lambda with a static IP has never been so simple 🍰'
cover_image: 'https://dev-to-uploads.s3.amazonaws.com/uploads/articles/88uovzvmgxnl2j8r3scw.png'
description: 'Learn how to deploy a Lambda with a static IP (for whitelisting concerns) and perform NodeJS SFTP operations using this Lambda.'
tags: serverless, sftp, aws, Lambda
series:
canonical_url:
---

## TL;DR <br>
🧠 Learn how to deploy a **Lambda with a static IP** (for whitelisting concerns)
⚡ Perform **NodeJS SFTP operations** using this Lambda. <br>

---

Your Serverless application might need to connect to a partnet server that requires IP whitelisting.
We will use the [Serverless Framework](https://www.serverless.com/) to deploy our Lambda function and the **Typescript AWS CDK** to create the infrastructure needed to give our Lambda a static IP 🔒. <br>
If you are not familiar with these libraries, I recommend you check out this [Ultimate Serverless DevX: Serverless Framework x CDK blog post](https://dev.to/kumo/the-ultimate-serverless-devx-serverless-framework-aws-cdk-and-typescript-3694) to learn how to conciliate the two. 🤝

Moreover, we will use the NodeJS [ssh2-promise](https://www.npmjs.com/package/ssh2-promise) package to perform SFTP operations in our Lambda function. 💽

This is what we will build 👷:

![Static Lambda IP Architecture](./assets/architecture.png 'Lambda with static IP Architecture')

Let's get to work 💪🔨!

# 1️⃣ Create a VPC hosted Lambda function with a static IP

## ☁️ VPC, NAT Gateway and Elastic IP


First, we need to create a VPC with a public subnet and a private subnet. The private subnet will host our Lambda function and the public subnet will host our NAT Gateway. <br>

```ts
const vpc = new Vpc(stack, 'Vpc', {
	vpcName: 'vpc',
	natGateways: 1, // 👈 Automatically creates an Elastic IP
	maxAzs: 1, // 👈 Use more if you need high availability
	subnetConfiguration: [
		{
		name: 'private-subnet-1',
		subnetType: SubnetType.PRIVATE_WITH_NAT,
		cidrMask: 24,
		},
		{
		name: 'public-subnet-1',
		subnetType: SubnetType.PUBLIC,
		cidrMask: 24,
		},
	],
});
```
> 💸 **Pricing Disclaimer**
> This architecture will cost **some money**. The NAT Gateway runs on a EC2 instance which will cost you **around 30$/month**. This price is multiplied by the number of AZs you use. 
 
 I also recommend that you use CloudFormation Outputs to store the IDs of the resources you will need later on. We want to slap that *✨clean IaC code✨* in our PR 🫱.
 
 ```ts

export const vpcSecurityGroupOutputId = 'SgOutputId';
export const vpcPrivateSubnetOutputId = 'VpcPvSubOutputId';
...

const privateSubnets = vpc.selectSubnets(
	{ subnetType: SubnetType.PRIVATE_WITH_NAT }
);
export const [privateSubnetId1] = privateSubnets.subnetIds;

// 👇 Outputing the IDs of the resources in case another stack needs them
new CfnOutput(stack, vpcSecurityGroupOutputId, {
	value: vpc.vpcDefaultSecurityGroup,
});
new CfnOutput(stack, vpcPrivateSubnet1OutputId, {
	value: privateSubnetId1,
});

```

## ⚡ Deploy the Lambda function

Now that we have our VPC, we can deploy our Lambda function, using the Serverless Framework. 

```ts
export const staticIpLambda = {
  timeout: 15,
  handler: getHandlerPath(__dirname),
  vpc: {
    securityGroupIds: [ stack.resolve(vpc.vpcDefaultSecurityGroup) ],
    subnetIds: [ stack.resolve(privateSubnetId1) ],
  	},
};
```

You can also check out this [Serverless Framework x CDK article](https://dev.to/kumo/serverless-framework-aws-cdk-1dnf) to learn how to leverage the `stack.resolve()` method 🧑‍🚀.

And bam! 🎉 Our Lambda function is deployed in our private subnet. <br>
All the outbound traffic from our Lambda function will now go through the NAT Gateway, which will use the Elastic IP we created earlier. 🚀

---

You can retrieve the Elastic IP of your NAT Gateway using the AWS Console.
Go to the VPC service and click Elastic IPs. You should see the Elastic IP created by the CDK. <br>

![Elastic IP](./assets/elasticIp.png 'Elastic IP on the AWS Console')

Now just message your partner and ask them to whitelist this IP. 📩

# 2️⃣ Perform SFTP operations in your Lambda function

> ❓ This part is actually optional. 
> We will cover a very specific SFTP use case and a problem I encountered while using the `ssh2-promise` and the `serverless-esbuild` plugin.

Now let's talk about a very specific use case: **performing SFTP operations in your Lambda function**. <br>
I wanted to share with you this use case because I came across weird issues with the `ssh2-promise` package not being correctly bundled into my Lambda `.zip code`. 🤯

> 🔒 **SSH Key** 
If your SFTP partner requires a SSH Key, I recommend you to store the key in AWS Secrets Manager. You can then retrieve it in your Lambda function using [Middy Secrets Manager middleware](https://middy.js.org/docs/middlewares/secrets-manager/). It will provide the key as a string in your **Lambda function context**. 🤓

## 💽 Store the file you want to send in your Lambda's `/tmp` folder

The `ssh2-promise` package will need file system access to send your file. One way to achieve that in a Lambda function context is to leverage the `/tmp` folder.
This folder is writable and will be deleted when your Lambda function is terminated.

> 🚧 **Warning**
> The `/tmp` folder is not persistent. If you want to keep track of the files you've sent you should also use S3. 📦
> Also be aware that the `/tmp` folder is shared between successive Lambda invocations in the same execution environment. Use a unique file name to avoid any bugs 🐛.

This is what the final code looks like with the `ssh2-promise` package:

```ts
fs.writeFileSync('/tmp/myFile.txt', 'Hello World!');

const sshPrivateKey = context["SSH_PRIVATE_KEY"]

const ssh = new SSH2Promise({
	host: 'sftpIpAddress',
	username: 'sftpUsername',
	privateKey: sshPrivateKey,
	port: "SFTP_PORT",
});

await ssh.connect();
const sftp = ssh.sftp();


await sftp.fastPut('/tmp/myFile.txt', "distant_name.txt");

await ssh.close();
```

## The catch 🎣

If like me you are using the `serverless-esbuild` plugin to bundle your Lambda function, you might encounter some weird issues. 
The `ssh2-promise` package is not correctly bundled into your Lambda function. 🤯

To solve this issue, I first patched the `serverless-esbuild` plugin to allow yarn3 module bundling exclusion. I want the `ssh2-promise` package **NOT** to be bundled by esbuild. 

The patch is available here as a [github gist](https://gist.github.com/valentinbeggi/07041590f2c42f93def7beed83466314)

Then, I had to add the `ssh2-promise` package to the `externals` section of my `esbuild` config. 

```ts
// serverless.ts

const serverlessConfiguration = {
  ..., // 👈 Your other serverless config
  custom: {
    esbuild: { ...esbuildConfig, external: ['ssh2-promise'] },
  },
};
```

Your Lambda function should now be able to perform SFTP operations. 🚀

---

Thanks for reading! 🙏

If you have any questions or feedback, feel free to reach out to me on [Twitter](https://twitter.com/valentinbeggi) or ask a question in the comments below. 🥸

Also check out my [Dev.to](https://dev.to/valentinbeggi) for more articles about AWS, Serverless and Cloud Development. 📝