---
type: posts
title: Installing a Developing Feature
author: Amutheezan Sivagnanam
category: Internship  
date: 2016-10-06
tags:
- wso2
- devfeature

---

The content did not fully belong to me, I just do some value addition to reference by explaining my project scenario
with this. For clear understanding and deep learning go through references.

Carbon Features are set of Features which was developed for the support of WSO2 Products, these are installed by default
or manually. For manually installing those features we need to locate the location of repositories where those features
are located. There is a common location available for all repositories
in [online](http://wso2.com/projects/carbon/provisioning-wso2-carbon-with-equinox-p2/), you just need to specify those
repositories based on the carbon version of your product. This is for the people who want to access the features to do
things.

So in my project perspective, I need to improve the "HL7 Feature" which currently under development. For this,

1. First clone the repository containing the feature. For me it is carbon-meditation.
2. Go to the component of the relevant feature.For me it
   is ```carbon-mediation/components/business-adapters/ hl7/ org.wso2.carbon.business.messaging.hl7.common/``` - (
   Actually only this part of the main component is modifying so I need to build it only, but it is better to build hl7
   in case if we you modifying other or not). Build the HL7 "component"; component contains basic sources for the
   feature, about how it works and implementation, but features are consisting of several subparts that are bundle
   separately when building this only.
3. Then build ```feature```. this will make a bundle with full feature combined with necessary dependencies additional
   to bundle and make the feature finalized. For my case it is
   in ```carbon-mediation/features/business-adapters/org.wso2.carbon.hl7.feature/```

## **WHILE TESTING WITH**

When building **COMPONENT** and **FEATURE,** it will generate a zip file in the target of feature which is actually
bundling the whole feature with necessary dependencies and configuration files. Based on features we need to unzip the
zip file and add contents to the working server. In my case, it doesn't contain any config file so I just unzip and copy
the jar files to the ```/repository/components/dropins/``` and restart the relevant server. It will use the feature
which is stored in dropins by doing necessary actions. Actually, for my case, it will copy those jars
to ```/repository/components/plugins/```, but in case if any configuration found. It will do that also. By doing this we
can test the system.

## **[this is based on Inosh Goonerwardana's Mentoring]**

## **FOR FINAL DELIVERY**

For my project perspective, when building **COMPONENT** and **FEATURE**, It will make the HL7 feature available
at ```/home/user/.m2/repositories/``` (the place where all installed repositories installed, and this is the place where
maven lookup in search of repositories.).Like this based on your feature, you build the necessary part and make it
available at ```/home/user/.m2/repositories/```.

To make this accessible by desired feature installing system in Management console we need to add the particular
SNAPSHOT version/latest version of the required feature using the tag in the carbon-feature repository and build it.
When building it will add the SNAPSHOT/latest version of the feature to ```p2-repo```. Then by using local repositories,
we can install features [1](http://blog.kasunbg.org/2013/04/generating-wso2-p2-repo-for-features.html).

### **References**

1. [http://blog.kasunbg.org/2013/04/generating-wso2-p2-repo-for-features.html](http://blog.kasunbg.org/2013/04/generating-wso2-p2-repo-for-features.html)
