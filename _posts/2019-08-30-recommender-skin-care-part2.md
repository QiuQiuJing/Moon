---
layout: post
title: "Recommender System: Skin Care Products (Part 2)"
date: 2019-08-30
excerpt: "Content-based filtering and Collaborative filtering"
tags: [Content-based filtering, Collaborative filtering, Recommender system]
comments: false
feature: 'assets/img/part1'
---

In previous post, I've finished data collection and data cleaning. In this article, I will talk about how to build a hybrid recommender system with content-based filtering method and collaborative filtering method. 

## Content-based Filtering
Content-based filtering allows the discovery of similarity between products based on item content. Customers ususally have certain preference on skin care products. For example, if some users like to use un-fragranced facial cleansers, they are less likely to buy fragranced toners.

To build a content-based filtering system, we first create our own scoring system based on product **ingredients** and **price**. The features are:

<figure>
  <a href="/assets/img/8features.png"><img src="/assets/img/8features.png"></a>
  <figcaption><a>8 features for content-based filtering method</a></figcaption>
</figure>

For ingredient related features, first I search for lists of chemicals that provides certain types of effect. For example, glucose provides moisturizing effect, vitamin C provides whitening effect. Then, I compiled all chemicals in a table:

<figure>
  <a href="https://miro.medium.com/max/792/1*N2lHAHtuUV5vAwj7OmVRpA.png"><img src="https://miro.medium.com/max/792/1*N2lHAHtuUV5vAwj7OmVRpA.png"></a>
  <figcaption><a></a></figcaption>
</figure>

### (1) NLP
Since the ingredients of products could come in full chemical names or abbreviated forms, we use **NLP** to tokenise them. we have two approaches: 
* unigram 
* combination of unigram and bigram.

{% highlight python %}
#method 1 unigram
for i in range(len(feature_list)):
    chem = df2[df2.feature == feature_list[i]].chemicals
    l = []
    for c in chem:
        l=l+tokenizer.tokenize(c.lower())
        df_feature.chemicals_uni[i] = l
#method 2 combination of unigram and bigram
for i in range(len(feature_list)):
    chem = df2[df2.feature == feature_list[i]].chemicals
    l = []
    for c in chem:
        if ' ' in c:
            bigrams = ngrams(c.lower().split(), 2)
            l.extend(bigrams)
        else:
            l.append(c.lower())
    df_feature.chemicals_unibi[i] = l
{% endhighlight %}

After trying both approaches, we find that unigram is more suitable for some features while a combination of unigram and bigram is better for others. For example, *cleanliness* is better represented with the second approach.

<figure>
  <a href="https://miro.medium.com/max/2676/1*BpZ9l5ilYqbCXxHwxElPlg.png"><img src="https://miro.medium.com/max/2676/1*BpZ9l5ilYqbCXxHwxElPlg.png"></a>
  <figcaption><a>cleanliness score for combination of unigram and bigram</a></figcaption>
</figure>

Lastly, for *mildness*, we collect chemicals with soothing effect and chemicals with irritancy effects. Based on that, define a function to determine the mildness score of product. This also consider as a simple **sentiment analysis** for product ingredients.

{% highlight python %}
#part 2 sentiment analysis for soothing and irritancy 
#in this case, we will treat soothing as positive and irritancy as negetive
#use combination of unigram and bigram
def simple_sentiment(ingredient):
    # positive/negative 
    index1 = df_feature[df_feature['feature'] == 'soothing'].index.values[0]
    index2 = df_feature[df_feature['feature'] == 'irritancy'].index.values[0]
    positive = df_feature.chemicals_unibi[index1]
    negative = df_feature.chemicals_unibi[index2]
    # Count 
    positive_count = sum([1 for i in ingredient if i in positive])
    negative_count = sum([1 for i in ingredient if i in negative])
    # Calculate Sentiment Percentage 
    # (Positive Count - Negative Count) / (Total Count)
    return round((positive_count - negative_count) / len(ingredient), 6)
#mildness score 　
l = []
for ingre in df1.uni_bigram:
    score = simple_sentiment(ingre)
    l.append(score)
df1.loc[:,'mildness'] = l 
{% endhighlight %}

### (2) Similarity
From that, we construct features scores for all the products using **Jaccard similarity**. Note that the order of ingredients matters. The chemicals that appear first have higher concentrations in the products.

{% highlight python %}
#jaccard_similarity
def jaccard_similarity(list1, list2):
    s1 = set(list1)
    s2 = set(list2)
    return len(s1.intersection(s2)) / len(s1.union(s2))
#using different method for different features
feature_list_1 = ['moisturising effect','whitening','viscosity']
feature_list_2 = ['cleanliness','antioxidant','fragrance']
def ingre_similarity(product, feature, chemical):
    for e in feature:
        s = []
        length = len(product)
        n = df_feature[df_feature['feature'] == e].index.values[0]

        for i in range(length):
            score1 = jaccard_similarity(product[i][0:int(length/3)], chemical[n])  
            score2 = jaccard_similarity(product[i][int(length/3):int(length*2/3)], chemical[n])
            score3 = jaccard_similarity(product[i][int(length*2/3):length], chemical[n])
            score = 0.5*score1 + 0.3*score2 +0.2*score3   
            s.append(score)
        df1[e] = s       
ingre_similarity(df1.unigram, feature_list_1, df_feature.chemicals_uni)
ingre_similarity(df1.uni_bigram, feature_list_2, df_feature.chemicals_unibi)
{% endhighlight %}

Next, we are going to concatenate all the scores to form and a new dataset and normalised them to the range of 0 to 1.

{% highlight python %}
product_score1 = df[['full_name','brand', 'Category','rating','price_oz']]
product_score2 = df1[['moisturising effect','cleanliness','mildness','antioxidant','whitening','viscosity','fragrance']]
product_score = pd.concat([product_score1, product_score2], axis = 1)
#normalize tha score to the range of 0-1
def norm(series):
    l = []
    for i in range(len(series)):
        n = (series[i] - series.min()) / (series.max() - series.min())
        l.append(n)
    series = l
features = ['price_oz','moisturising effect','cleanliness','mildness','antioxidant','whitening','viscosity','fragrance']
for feature in features:
    norm(series)
    product_score[feature] = l
{% endhighlight %}

Finally, construct a content-based similarity matrix for products by using **Cosine similarity**.

{% highlight python %}
#exclude norminal features
content = product_score.iloc[:,4:11]
#content similarity matrix
sim_matrix = cosine_similarity(content)
content_sim = pd.DataFrame(sim_matrix, columns=content.index, index=content.index)
{% endhighlight %}

## Collaborative Filtering
Collaborative filtering is a method of making predictions about the interests of a user by collecting preferences information from many users.

In this part, we are going to use collaborative filtering method with user ratings. More specifically, we are going to find out the similarity between products from the user’s point of view. And it could recommend totally different products as compared to content based filtering. 

For example, users may not always like products with the most similar ingredients. A user could use a toner to sooth her skin, but use a moisturiser for its whitening effect. Or maybe users use two products have quite different ingredients but from the same brand.

First, we construct a user-product matrix that tells us all ratings from users for each product. We drop users who only bought one product.

{% highlight python %}
#constract a users vs product matrix, it contains rating products from users
#fill in 0 if no ratings
up = pd.pivot_table(review, index='user', columns='id', values='stars').fillna(0)
#find users who only purchase one product
l = []
for i in up.index:
    s = sum([1 for i in up.loc[i] if i != 0.0])
    if s < 2:
        l.append(i)
#drop
up.drop(index = l, inplace = True) 
{% endhighlight %}

Follow up, we are going to do model evaluation with **Surprise**. It allows me to have a better handling of my data and provides various ready-to-use algorithms for recommender system.

{% highlight python %}
#data preparetions
# set rating_scale param 
reader = Reader(rating_scale=(1, 5))
# The columns must correspond to user id, item id and ratings (in that order).
data = Dataset.load_from_df(review1[['user', 'id', 'stars']], reader)
#train test split
trainset, testset = train_test_split(data, test_size = 0.25)
{% endhighlight %}

After removing users who only rated one product. I have 44,359 reviews from 17,278 users, which means there are only 2.6 ratings per user in average. Next, construct a user-item matrix with user ratings as values. we need to deal with **Sparse matrix** problem. In other words, we need to fill in all the empty values in user-item matrix. The algorithms used are:
* **SVD**
* **SVD++**

### (1) SVD
SVD is short form for **Singular Value Decomposition**. It decreases the dimension of the utility matrix by extracting its latent factors.

{% highlight python %}
#Grid Search for the best parameters
param_grid = {'n_factors':[40, 60], 
              'n_epochs': [20, 40, 60],
              'lr_all': [0.002, 0.005],
              'reg_all': [0.1, 0.4, 0.6]}
gs = GridSearchCV(SVD, param_grid, measures=['rmse', 'mae'], cv=3)
gs.fit(data)
# best RMSE score
print(gs.best_score['rmse'])
# combination of parameters that gave the best RMSE score
print(gs.best_params['rmse'])
#run the SVD model
algo = gs.best_estimator['rmse']
algo.fit(trainset)
test_pred = algo.test(testset)
accuracy.rmse(test_pred)
{% endhighlight %}

Results are:
Algorithm | RMSE for Training Set | RMSE for Test Set
----------| ----------------------| -----------------
**SVD** | 1.143683 | 1.151032

### (2) SVD++
SVD++ is an extention to normal SVD. The difference is the extra implicit information. More specifically，user rates an item is in itself an indication of preference.

{% highlight python %}
#Grid Search for the best parameters
param_grid = {'n_factors':[40, 60], 'n_epochs': [20, 40], 'lr_all': [0.005, 0.007],
              'reg_all': [0.2, 0.4, 0.6]}
gs = GridSearchCV(SVDpp, param_grid, measures=['rmse', 'mae'], cv=3)
gs.fit(data)
# best RMSE score
print(gs.best_score['rmse'])
# combination of parameters that gave the best RMSE score
print(gs.best_params['rmse'])
#after great grid search
algo = SVDpp(n_factors = 40, n_epochs = 40, lr_all = 0.005, reg_all = 0.2)
algo.fit(trainset)
test_pred = algo.test(testset)
accuracy.rmse(test_pred)
{% endhighlight %}

Here are results for two method:
Algorithm | RMSE for Training Set | RMSE for Test Set
----------| ----------------------| -----------------
**SVD** | 1.143683 | 1.151032
**SVD++** | 1.145048 | 1.150734

From the table above, we find out scores are qiute close for. Thus, I choose SVD since it has a simpler formula and takes less time to run compare to SVD++.
{% highlight python %}
up_matrix = up.as_matrix()
#Performs matrix factorization of the original user item matrix
U, sigma, Vt = svds(up_matrix , k = 40)
sigma = np.diag(sigma)
#reconstruct the matix
all_user_predicted_ratings = np.dot(np.dot(U, sigma), Vt) 
#Converting the reconstructed matrix back to a Pandas dataframe
item_based = pd.DataFrame(all_user_predicted_ratings, columns = up.columns, index=list(up.index)).transpose()
{% endhighlight %}

Next, use the same method as we did for content-based filtering, construct a similarity matrix for collaborative system.

## Aggregation and Recommendation
Finally, combine two methods to build a hybrid recommender system. To aggregate the results from both systems, we create a recommender function. 
1. Input the product you like and the category you are interested
2. Rank all the products in the category users are looking for for both content based and collaborative systems. 
3. Add the two ranks together and rearrange them in an ascending order. If the rankings are the same for two products, we will take into account the repurchase rate. A product that has a higher repurchase rate will receive a higher ranking compared to another.
4. Show top 10 recommendations.

{% highlight python %}
def recommender(cat, product_name):
    product_id = df[df.full_name == product_name].index[0]
    #index of products from desired category
    cat_in = df[df.Category == cat].index    
    #rank product by similarity, lower the ranking number, higher the similarity
    content_rank = content.iloc[product_id,cat_in].rank(ascending=False, method='min')
    collab_rank = collab.iloc[product_id,cat_in].rank(ascending=False, method='min')
    #aggregate two rankings to get overall rankings
    rank = content_rank + collab_rank
    #sort rankings with ascending orders, products on top will most likely be recommended
    rank = rank.sort_values()[1:]
    #for products have same rankings after aggragation, we will take into account the repurchase rate.
    re_one = rank[rank.duplicated()]
    re_all = rank[rank.duplicated(keep=False)]
    
    for i in re_one:
        list_index = []
        list_num = []
        for e in re_all.index:
            if i == re_all[e]:
                list_index.append(e)
                list_num.append(int(e))
        r = repurchase.loc[list_num,'rate'].rank(ascending = False,method = 'min')
        rank[list_index] = rank[list_index] + list(0.01*r)              
    #show the top 10 products' infomation  
    result = df.iloc[list(rank.sort_values()[:10].index)] 
    #show cosine similarity score for both content base and collaborative
    content_sim = list(content.iloc[product_id,list(result.index)])
    collab_sim = list(collab.iloc[product_id,list(result.index)])
    result.loc[:,'content_sim'] = content_sim
    result.loc[:,'collab_sim'] = collab_sim
    final = result[['full_name','brand','content_sim','collab_sim']]
    return final
{% endhighlight %}

Finally, I use Heroku to deply the recommender system.

<figure>
  <a href="/assets/img/heroku.png"><img src="/assets/img/heroku.png"></a>
  <figcaption><a href="https://skin123.herokuapp.com/home">Skin123!</a></figcaption>
</figure>

For more information, please check my [Github](https://github.com/QiuQiuJing/Capstone) post.



