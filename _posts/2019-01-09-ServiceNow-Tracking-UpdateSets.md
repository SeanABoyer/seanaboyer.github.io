---
layout: post
title: Tracking Update Sets and Relating them to Tasks
---
A lightweight application that tracks the environments an update set has been committed to. Capable of linking back to a production record (INC/Story/etc). Works with Single Update Sets, Merged Update Sets, and Batch Update Sets.

## Problem
Both at my previous company and my current company, the tracking of update sets has been difficult. Anytime the team size grows beyond 1-2 people it can get difficult to track where exactly an update set is. Throw in some update sets that need to be delayed for some reason and you might have to spend some time looking for the update set.


## Quick Solution OverView
I created a way for update sets to be linked to records in a production instance and be able to track each individual update set to see which instance it had been committed in. This is more of a bolton feature to track update sets, with the hopes of being able to use this to create an automated pipeline in the future.

The solution is on ServiceNow's Community Share by the name of [Update Set Tracking](https://developer.servicenow.com/app.do#!/share/contents/4545518_update_set_tracking?v=1.0&t=PRODUCT_DETAILS). More Information is provided here as well as post installation/configuration steps.


## In-Depth Solution OverView
I created a table to store the information between an update set and a task record, containing the minimum amount of fields as the source of truth for the data should be in the Update Set or the Task itself.

After that I created a few Scripted Rest Service for the sub production instances to hit and provide the data to update the update set tracking table I created. For each Scripted Rest Service, there is a Rest Message, with the idea of keeping these 1 to 1.

I then created a few business rules to manage when to fire off to hit the EndPoints, 2 on the Local Update Set table and 2 on the Remote Update Set table.
On the Local Update Set table, one is to document the creation of a new Update set and the other is to document an update to the update set.
On the Remote Update set table, one is to document the additon of a new environment and the other is to document the removal of a environment.

Both the scripted Rest Services and Busines Rules all point back to a UST script include. the UST script include is an extension of the USTCore Script include so that, if needed extra logic can be added without messing with core code.

## Moving an Update Set From Dev to Prod (With Pictures)
1. Create an Update Set in Dev. (Not linking to a Task)
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/Step1.PNG "Step1") 
As you can see, it is now being tracked in our production instance.
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/Step1a.PNG "Step1a") 
2. Complete the Update Set in Dev, then Retrieve the Update Set from Dev in Test and Commit the Update Set in Test.
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/Step2.PNG "Step2") 
As you can see, the production update set tracker record has been updated to contain both the dev and test environment in the ServiceNow Environment Field.
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/Step2a.PNG "Step2a") 

3. Retrieve the Update Set form Test in Prod and Commit the Update Set in Prod.
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/Step3.PNG "Step3") 
As you can see, the production update set tracker rcord has been updated to contain all environments the update set has traveled through in the ServiceNow Environment Field.
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/Step3a.PNG "Step3a") 


## Changing an Update Set name. (With Pictures)
Updating the Update Set name will update the update set tracker record.

Old Name Update Set in Dev
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/OldDev.PNG "OldDev") 
Old Name Update Set in Prod
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/OldProd.PNG "OldProd") 
Here is the same update set with a name changed.
New Name Update Set in Dev
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/NewDev.PNG "NewDev") 
New Name Update Set in Prod
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/NewProd.PNG "NewProd") 


## Reverting/Back Out an Update Set. (With Pictures)
In some cases, you need to revert (or back out) an update set. But you still want to track that.

Before Back out
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/PreRevert.PNG "PreRevert") 
After Back Out
![alt text](https://raw.githubusercontent.com/SeanABoyer/seanaboyer.github.io/master/img/posts/UpdateSetTracker/PostRevert.PNG "PostRevert")