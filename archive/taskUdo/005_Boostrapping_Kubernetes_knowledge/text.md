## Introduction
In my previous [post][1], I discussed my assertion that with new Cloud technologies, it made more sense to develop web service backends that were targeted at individuals, i.e. Personal Backend Services, as opposed to Shared Backend Services, and this is what I planned to use for taskUdo. However, I also admitted that this was not based on practical experience of either the technologies or costs.  

Clearly, lack of practical knowledge was something that needed rectifying, so my plan was to essentially:
1. Bootstrap my [Kubernetes][2] knowledge by undertaking some basic training focused on learning how to deploy applications to it.
2. Use a learning reinforcement exercise/test, where I would attempt to move a non-trivial application, in this case[ownCloud][3], onto Kubernetes, running:
	- Firstly, on a local, testing cluster.
	- Secondly, in a production cloud environment, for which I had chosen, no surprises, [Google Cloud Platform][4].

Here's what I did and found for the first part of this.

## Bootstrapping Kubernetes knowledge
My normal approach with bootstrapping skills is possibly a bit old fashioned. Even with bleeding edge stuff, it's usually Amazon, find a couple of books and dig in. This generally, I think, works well enough because of the greater effort involved in publishing books (even if they are not in dead tree format), tends to result in information that is of a bit greater depth, higher production standards and more long term value (usually anyway). 

This time though I decided not to go with that approach. As yet, there is a paucity of books on Kubernetes, and, from what I can tell, the existing ones are more focused on building clusters and running more complex backend services than I hope will be necessary. Instead, I decided to try the somewhat radical approach, for me anyway, of directly going with the OpenSource project's documentation and training materials.
![Relying on docs from projects where a lot of them make their money through books and support contracts can be a bit of a risk.][image-1]

### Plan
I decided to bootstrap my Kubernetes knowledge by working through the Kubernetes list of official tutorials from [here][5].

I'd start with their zero install[Kubernetes Basics, interactive demo/tutorial][6] [^1], since not having to install anything, was one less thing for me to get wrong, or distract myself with.  

Once through that, then I was going to install onto my Mac a local test cluster setup by following the instructions in the [Hello Minikube][8] tutorial to install minikube.

With minikube in place,  I'd then chug through the rest of the tutorials (but omit the full on [Online Training Course][9]) and hopefully be competent enough at the end of them to know where to start with subsequently bringing up ownCloud.

### Observations and thoughts
#### Getting a local Kubernetes test/training cluster setup working
[minikube][10] is the Kubernetes project current recommended front-end tool for creating and managing local test clusters.

The [Hello Minikube][11] tutorial itself steps you through installing minikube and getting everything else you need installed to set up a local Kubernetes test cluster and its tools. 

On macOS, this list ends up being:
- [Homebrew][12] 
- [Docker for Mac][13]
- [The Docker Machine Xhyve Drive][14]
- The [minikube software][15] itself.
- The main Kubernetes cluster interaction client software kubectl

It's not a huge list, but it's coming from a variety of different providers, so the actual installation process can be a bit disjointed. However, I think if you follow the steps in the tutorial carefully you should reliably end up with a working local Kubernetes test cluster without too many problems, I did anyway.

Having dealt with both commercial and OpenSource installations of similar complexity, I have to admit that I was actually very relieved by how smooth the whole installation and setup process was (and how well it has performed since).
![Perhaps not quite as spectacular, but we have Kubernetes lift off ... hopefully,  nothing will blow up ...][image-2]

#### Using local minikube and the local Kubernetes test
Whilst getting minikube installed and working was straightforward, and I haven't encountered any major problems with it, I have noticed a few wrinkles along the way.

##### Recommend allocating more resources to minikube
I'm not sure precisely which one it was, possibly the [Connecting a Front End to a Back End Using a Service][16] tutorial, but one of the tutorials wouldn't run with the default minikube configuration because the resource configuration was too small (you can see the defaults with `minikube start --help`).  

The giveaway that you might be bumping into a similar problem is when `kubectl describe pods your_pod_that_is_not_starting` mentions in its output that it can't schedule it.

My trusty laptop could take it [^2], so I fixed this by bumping up the defaults using `minikube config set` to allow it to use all four cores, doubled the amount of RAM because and at the same time I told it to use the `xhyve` driver by default, as I was getting bored of typing it when starting minikube.

`minikube config view` now shows:

	- memory: 4096
	- vm-driver: xhyve
	- WantReportError: true
	- cpus: 4

##### Bug (maybe), minikube Persistent Volume Claims not picking up manually provisioned volumes
One of the things that are frequently an application requirement is to be able to have a volume that does not get thrown away between invocations of your application.  For instance, such a volume might hold a database of users and their passwords that have been added to the system via its web interface, or some other similar data. 

If you are working your way through the Tutorials, the first time you bump into a demo of this is with [Running a Single-Instance Stateful Application][17]. In this example, the intent is to bring up a MySQL DB with the DB's files stored on a Persistent Volume. Unusually, the example expects you to be running on Google's Cloud Platform and because they use Google's `gcloud` tool to create the volume and you are left a bit in the dark if you are using anything else.

With a bit of the research, the answer to how to create the equivalent volume in minikube is actually contained in the docs [Tasks-\>Configuring a Pod to Use a PersistentVolume for Storage][18] article. In this article, they show how to write a file to a volume, and then have one of the pods serve it up using minikube.

Except, in the last couple of weeks that example has stopped working because the system has, with possibly the latest update, started ignoring manually created Persistent Volumes and is instead[always dynamically creating fresh, temporary volumes][19]. 

The workaround is to annotate the [PersistentVolumeClaim with an empty storage type][20], which disables fallback automatic volume provisioning for this claim and causes it to correctly bind to the manually created one instead.

So for the [example][21], the workaround PersistentVolumeClaim looks like this:

	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: task-pv-claim
	  "annotations": {
	        # Prevent creating new volumes on the fly by not specifying.
	        "volume.beta.kubernetes.io/storage-class": ""
	  }
	
	spec:
	  accessModes:
	    - ReadWriteOnce
	  resources:
	    requests:
	      storage: 3Gi
 

##### Bug, leaks IP addresses when deleting cluster
One of the things I eventually noticed after experimenting with using `minikube delete` as a way to completely reset the cluster, was that it would leak IP addresses [^3].  Basically, after each `minikube delete` operation, you would find `minikube ip` returns an IP address that is incremented from its previous one.  As of the 14th of March 2017,  I think this remains broken, there is certainly a ticket open for the issue on GitHub [here][22].

In a meantime, the workarounds are:
1. Live with it and use `minikube delete` as little as possible.
2.  You can manually reset the numbers by removing a couple of system files, as a suggested in the cross-linked [comment][23] for a different ticket (708). I used this successfully, but be careful with this option as **it's all too easy to accidentally remove the wrong files and end up with a broken system**. 

#### Tutorials
I'm not an IT admin, full on Linux hacker or Jedi programmer in any way shape or form. However, I have been in the industry for quite a while and picked up a few things along the way.  Specifically, I have been playing with Linux from mid 90's, my team at ARM developed a large distributed Build System and I have quite a lot of background knowledge about services and cloud computing as a result of this.

 So not a blank slate, I'll usually know roughly what to search for, who to ask, but no previous practical experience and this is what I thought about Tutorials on the Kubernetes site.

##### Content and pace
I found the tutorials informative, stepped me through at about the right pace and, covered almost everything I needed to know to find my way around and get started using Kubernetes for application development.

The only two technical issues I had with the tutorials were related to using minikube I have mentioned those already.

##### Coverage,
Ultimately I ended up skipping a few of the tutorials, not because I don't think they would be useful for most folks, but because after I had glanced through them, they were more complicated than I'm hoping to need for taskUdo's backend approach.  Specifically, I haven't really looked at:
- [Running ZooKeeper, A CP Distributed System][24].
- [AppArmor][25]
- [Setting up Cluster Federation with Kubefed][26]

Other than that, and after tackling getting ownCloud going locally and on Google's Cloud Platform, I thought there was one obvious omission from the content, and FSFA DSF FD F really nice to have improvements.

For me, the only major omission in the tutorials that I noticed, so far anyway, was the lack of coverage of the use of Secrets. If you search the site there is good information's present on how to create and use them in the Glossary [here][27], I'd argue the need to use store use and store passwords if likely to be a basic requirement for most of us, so probably should have been covered. 

Beyond that, I know that some of it is not a core concern, and some of it is because of a need for vendor neutrality, but  it would have been nice if there was:
- Some examples of how to work with Google Cloud Platform, Amazon Web Services, Azure etc to hand.  Coverage of such things as Ingresses, static IP address provisioning, Persistent Volumes,  all of which have an element of cloud vendor specificity. 
- A bit more guidance, or pointers to where to look when it came to figuring out why it wasn't working, e.g. [Tasks-\> Monitoring, Logging, and Debugging][28]
- Working with containers, particularly what happened when using exec to permissions and entry points.
- Getting encryption and SSL working, ideally something featuring the use of [Let's Encrypt][29]
	  
#### Tips
If you're planning to use either Kubernetes or minikube here are a few things that you might find useful:
1. If it is an option for you, then make sure you have extended command line completions installed. It'll help you explore command and sub-command options as well as save you a lot of typing. You can find the official how to do this [here][30] and I mention how to do it in a more general macOS context over [here][31].
2. Familiarise yourself with the contents of [Tasks-\>Debugging Init Containers][32] and [Support-\>Troubleshooting Applications][33] (`kubectl describe` and `kubectl log` are your friends).
3. Similarly `kubectl exec -it bash pod_name`is very useful for logging into containers in your pod and  looking at what the heck is happening inside of them.
4.  Most containers are very bare bones. If you log into them with `kubectl exec`, then you may find that many of the normal Linux tools are not available. Assuming the container is Debian based (which a lot of the are) then `apt-get update` followed by  `apt-get install tool_you_want` (where tool\_you\_want is usually vim) is helpful to know if you've not used Linux much
5. The Kubernetes site search works really well.  I've only really found the need to resort to more general searches/StackOverflow when I've been trying to figure out how to make something work with Google's Cloud Platform, rather than with Kubernetes per se. So for the most up to date and relevant information on Kubernetes, I would recommend using the kubernetes.io site search by default.
6. There is a lot of information in the kubernetes.io site and some of the best bits of it are not always  where you might expect to find them. In addition to making sure to use the site search, I think it's probably a good idea to at least know roughly what is available under the other documentation areas. I think in particular you might want to have a browse around:
	- [Tasks][34].
	- The contents of the [Reference Documentation -\> Glossary][35].
	- The list of [full sample application deployments][36].        

### Conclusions
At the outset, the plan was to attempt to bootstrap a reasonable level of competency with Kubernetes by following the Tutorials and documents on the [kubernete.io][37] website.

It appeared as a reasonable approach, as documentation and training are evidently something that is important to the project. For instance, if you just want to quickly try the system out and you go to the project's [Home page][38]. Then there front and centre is a link to a ["Try Our Interactive Tutorials"][39].  Whilst at the other end of the scale, if you follow the "Documentation" link from the Homepage, you'll find the [Tutorial's page][40], which in turn links to a free, and I would guess comprehensive, [Udacity online training course][41] led by some of the leading lights in the Kubernetes community. 

It's was a few weeks ago now that I completed my training process using the tutorials. Since then I've gone on to get ownCloud running on minikube and on Google's Cloud Platform (more on this in subsequent posts). Within the limitations I've already mentioned, I think there is more than enough good quality information on the website for most folks to get themselves up and running with it.

Overall the docs, like the rest of the project, are in my opinion very good for such a large complicated system. The strongest testament to their quality is perhaps that I have felt no need to purchase additional training materials from anywhere else.

[^1]:	**NB: As of 2017/03/12 this option works fine in Chrome and Firefox, but seems to be [broken in Safari][7]. **

[^2]:	Late 2013 MacBook Pro, i7 with 16GB of RAM

[^3]:	This was a bit irksome, especially when working with ownCloud because its security setup meant I needed to be able to pre-configure the IP address with ownCloud in order to be able to log into it.

[1]:	https://taskudo.info/blog/plan-validation/
[2]:	https://kubernetes.io
[3]:	https://owncloud.org
[4]:	https://cloud.google.com
[5]:	https://kubernetes.io/docs/tutorials/
[6]:	https://kubernetes.io/docs/tutorials/kubernetes-basics/
[7]:	https://github.com/kubernetes/kubernetes.github.io/issues/2712
[8]:	https://kubernetes.io/docs/tutorials/stateless-application/hello-minikube/
[9]:	https://www.udacity.com/course/scalable-microservices-with-kubernetes--ud615
[10]:	https://github.com/kubernetes/minikube
[11]:	https://kubernetes.io/docs/tutorials/stateless-application/hello-minikube/
[12]:	https://brew.sh/
[13]:	https://docs.docker.com/docker-for-mac/
[14]:	https://github.com/zchee/docker-machine-driver-xhyve
[15]:	https://github.com/kubernetes/minikube
[16]:	https://kubernetes.io/docs/tutorials/connecting-apps/connecting-frontend-backend/
[17]:	https://kubernetes.io/docs/tutorials/stateful-application/run-stateful-application/
[18]:	https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
[19]:	https://github.com/kubernetes/kubernetes.github.io/issues/2803
[20]:	https://kubernetes.io/docs/user-guide/persistent-volumes/#dynamic
[21]:	https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
[22]:	https://github.com/kubernetes/minikube/issues/951
[23]:	https://github.com/kubernetes/minikube/issues/708#issuecomment-255120598
[24]:	https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/
[25]:	https://kubernetes.io/docs/tutorials/clusters/apparmor/
[26]:	https://kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/
[27]:	https://kubernetes.io/docs/user-guide/secrets/
[28]:	https://kubernetes.io/docs/tasks/
[29]:	https://letsencrypt.org/
[30]:	https://kubernetes.io/docs/user-guide/prereqs/
[31]:	https://taskudo.info/blog/installing-enhanced-bash-autocompletion-on-macos-aka-restoring-command-line-mojo/
[32]:	https://kubernetes.io/docs/tasks/debug-application-cluster/debug-init-containers/
[33]:	https://kubernetes.io/docs/user-guide/application-troubleshooting/
[34]:	https://kubernetes.io/docs/tasks/
[35]:	https://kubernetes.io/docs/reference/
[36]:	https://kubernetes.io/docs/samples/
[37]:	https://kubernetes.io/docs/tutorials/
[38]:	https://kubernetes.io/
[39]:	https://kubernetes.io/docs/tutorials/kubernetes-basics/
[40]:	https://kubernetes.io/docs/tutorials/
[41]:	https://www.udacity.com/course/scalable-microservices-with-kubernetes--ud615

[image-1]:	./assets/Crazy_fool.jpg "BA Baracus meme, being slightly snarky about OpenSource documentation"
[image-2]:	./assets/DraggedImage.png