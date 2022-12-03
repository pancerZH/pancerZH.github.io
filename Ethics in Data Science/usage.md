---
sort: 4
---

# Outputs and Usage

## Interpretation of Outputs

Basically, the output of the model is a number of contents recommended for a certain user. Here the contents could be news, videos, musics, goods or posts. These contents are generated from the procedure I have mentioned in last chapter, which means they have gone through the filtering system to meet requirements given in past chapters. 

In a more deatiled manner, for a single time of recommendation, there could be like 10 news articles for the user. Among these 10 articles, there could be like 8 articles the user might be interested in, according to the user's browsering history and interest tags, if any; another 2 articles could be designed to be recommended to make the user see different contents, or expand the user's view on purpose. These 2 articles may be something that the user is not interested in, or has never expected before. Among all the 10 articles, the model has checked them to exclude what the user has marked as "not interested". In a statistic view of the complete recommendation results around users, the model would also try to make each news article has the same or close chance to be recommended, unless there are many enough negative feedbacks pointing to certain articles.

In a word, compared with traditional recommendation system, my model would not focus on accuracy solely, but also try to reach a just and fair goal of recommendation results, in both views of a single user and the whole system. 

## How the Model Increase Justice

### By making users expand their views and hearing different voices

In last chapter, I have talked about the unfaireness and unjustice of traditional recommendation system to keep users in their own loops. Some discussion of measures my model would take to response to these unfaireness and unjustice has also been given. Here I would dive deep to show a context.

We all know resources on earth are not distributed equally. People living in developed countries or big cities may contact more things and more ideas, compared to people living in less developed areas. With the same start points, a traditional recommendation system would naturally recommend a broder range of contents to the former people, and people with fewer resources tend to be locked in information cocoon much more easily. But with the help of my model to break information cocoon, people in less developed areas could get rid of their limition by their own life, and get the opportunity to explore things even they have never seen or imagined. This is due to the designed 20% of new contents recommended to users in the model, which is responsible for expanding users' views. This measurement, in fact, makes it more just for people in different areas with different resources to explore contents on the Internet. It is also clear that the recommendation system I designed would not solve the inequality from its root, but it at least makes things better.

### By making contents equal chance to be recommended

In a field, there are always a number of newcomers for contents generation. They could be a new editor, a new business, a new artist and many other roles. They are new to platforms and their contents tend to be ignored by other users, without some help from the recommendation system to bring their contents to other users. This is exactly what my model would do: by making all contents enjoy equal or nearly equal chance to be displayed to users, it could be more fair and just for these newcomers compared to a traditional recommendation system would do. On the other hand, if some contents are really bad, they would be less likely to be recommended to users by my model. In this way, high quality contents would not be ignored for starting at a low point, and low quality contents would not be always recommended even they are from some super star users.

### By making a quick respond to user's behavior

It is important to confirm one basic concept: users hope that the recommendation system make them happier. So if some recommendation results make some users happy while make other users sad, this is unfair and not we expect to see. In this way, the quick response of the model is pretty critical. Users who have experienced abortion or losing children wish not to see child products being recommended to them, users who have bought some goods wish not to see advertisements from the same or similar goods, and users who used to be in a war and wish not to see war reports: their willings must be respected and responded. The common unfairness and injustice in traditional recommendation systems are partially from the ignorance, to which my model would make a proper response by excluding these contents from recommendation lists.

## Situations to Update the Model

### When user quits

The model heavily depends on users' behavior, which means one recommendation list for a single user could be generated from all other users by visiting their data. It is then obvious that something should be done when some users or even a single user asks to stop using his or her data. In this scenario, the model must be re-trained with remaining data. The thing is, the model is partialy dynamic, which means it would read current dataset each time making a recommendation. While there are still parts relying on the whole original training dataset. When deleting users' data(behavior data, interest tags and feedbacks) without taking any action, the first part would work fine, as if these data never appears, while the second part is still relying on these deleted data, even indirectly. Their effects are still remaining in the recommendation system. 

Also, there should be a balance between retraining and using the model. In a common recommendation system, there could be millions of users in it, and there could be tens or hundreds of users asking to delete their data every day. It would be impractical to retrain the model each time a single user's data has been deleted. I plan to decide a retraining period, like midnight of each day, or another certain time point in a time interval. The user who chooses to delete the data from the system would be informed about this detail.

### When users' interests change

It is common that something is on trend in a certain period, and people could be no longer interested in it quickly. If the recommendation system still recommends a lot of related contents, people would get bored and less interested in the platform. On the other hand, the platform should not monitor what is on trend, because the platform should always keep nurtural. In this way, the best solution could also be regular update or retraining of the model.

This method could be combined with the former one, which also requires retraining of the model. This is the key point to keep the model always up-to-date and know what users' interests are.

### When more filtering systems are required

Different filtering systems, including quick response system and weight adjustment system, have been talked about. And I have mentioned that some more data beyond users' behavior data would be collected to support these systems. In the future, it is very possible that there are more requirements for filtering recommendation results, in which scenario it is unavoidable to collect more data from users.

The collection of new data is in the view of dimension, instead of quantity. In other words, I am not going to collect data from more users outside the system involving the model(there is none in fact), instead I may ask for more information from existing users. This behavior would require users' consents, obviously. However, it could also be possible that some of users refuse to give their consents. In this way, the additional filtering systems could be designed as optional: for users who do not give their consents, their recommendation results would not go through the specific filtering system.

### When not allowed to use others' behavior data

Just as I said above, my recommendation system relies on other users' behaviors to generate recommendation list for a certain user. There are also other recommendation systems that do not rely on users' behavior to make recommendation. So if one day behavior data is no longer allowed to be used for recommendation, the current model should be ceased and perhaps we can turn to another valid model.