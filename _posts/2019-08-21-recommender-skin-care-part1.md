---
layout: post
title: "Recommender System: Skin Care Products (Part 1)"
date: 2019-08-21
excerpt: "Web scraping and data cleaning"
tags: [web scraping, data cleaning, introduction]
comments: false
feature: 'assets/img/part1'
---

## Introduction
A basic skin care routine involves applying products such as facial cleansers, toners and moisturisers. However, many users face problems choosing products that suit them. 
In this post, I will bring you through how to build a recommender system based on:
* **content-based filtering method**
* **collaborative filtering method**

Using data from sephora.com, the system recommends a skin care product based on products that a user has liked, or what category of product the user is presently searching for.


Here is the outline: 
1. **Web scraping and data collection**
2. **Data cleaning and pre-processing**
3. **Content-based filtering**
4. **Collaborative filtering**
5. **Aggregation and recommendation**

In this post, I will walk through the first part and second part.



## Web craping and data collection
All data are scraped from [sephora.com](https://www.sephora.com). There are two parts of information are needed to build recommender system:

* **Product basic information**: includes basic product information such as categories, brands, product names, ingredients, prices, sizes, ratings, descriptions and pictures
* **User reviews**: includes user feedbacks for all products such as user names, ratings, reviews, time and helpfulness

I use [Selenium with python](https://selenium-python.readthedocs.io/), because web pages are not static and need to interact with them.

### (1) Urls for products
In this post, I choose three categories for skin care products, which are facial cleanser, toners and moisturisers. First, in each category, 
I scarp links for all selected products and create a dataframe with two features:
* `Category`: Three product categories
* `URL`: Product urls

{% highlight python %}
#step 1 
#Get url for product in each category
driver = webdriver.Chrome('./chromedriver')
#Categories include facial cleanser, toner and moisturizer
productcat = ['face-wash-facial-cleanser', 'facial-toner-skin-toner', 'moisturizer-skincare']
#create a dataframe
df = pd.DataFrame(columns=['Category', 'URL'])
#urls for all products
for cat in productcat:
    driver.get('https://www.sephora.com/shop/'+cat+'?pageSize=300')
    time.sleep(1)
    elem = driver.find_element_by_tag_name("body")
    #scroll page down to deal with lazy-load webpages
    no_of_pagedowns = 10
    while no_of_pagedowns:
        elem.send_keys(Keys.PAGE_DOWN)
        time.sleep(0.2)
        no_of_pagedowns-=1
    #find url
    pi = driver.find_elements_by_class_name("css-ix8km1")
    producturl = []
    for a in pi:
        subURL = a.get_attribute('href')
        producturl.append(subURL)    
    dic = {'Category': cat, 'URL': producturl}
    df = df.append(pd.DataFrame(dic), ignore_index = True)
{% endhighlight %}

### (2) Product informations
Second, using products urls, I scarp basic informations for all products. The features are 
* `brand`: Product brand 
* `name`: Product name
* `rating`: Overall rating for products
* `price`: Product price (us dollars) 
* `descriptions`: Product description
* `ingredients`: Product full ingredients, a list of chemicals 
* `pic`: Urls for product pictures 
* `productsize`: Product size (oz)


{% highlight python %}
#step 2
#scrape product brands, names, ratings, prices, descriptions, ingredients and pictures
df_info = pd.DataFrame(columns=['brand', 'name', 'rating', 'price', 'descriptions', 'ingredients','pic','productsize'])
df = pd.concat([df, df_info], axis = 1)
driver = webdriver.Chrome('./chromedriver')
for i in range(len(df)):
    driver.get(df.URL[i])
    time.sleep(5)
    #close ads window
    try:
        driver.find_element_by_class_name('css-fslzaf').click()
    except ElementNotVisibleException:
        pass
    except NoSuchElementException:
        pass
    #product brand and name
    item = driver.find_element_by_class_name('css-a1jw00').text.split('\n')
    df.brand[i] = item[0]
    df.name[i] = item[1]    
    #price
    df.price[i] = driver.find_element_by_class_name('css-14hdny6').text
    #descriptions
    df.descriptions[i]= driver.find_element_by_class_name('css-1rny024').text
    #one page down
    elem = driver.find_element_by_tag_name("body")
    elem.send_keys(Keys.PAGE_DOWN)
    time.sleep(1)
    #ingredient
    #we need to click ingredient tab in product infomation table to get the text
    driver.find_element_by_id('tab2').click()
    df.ingredients[i] = driver.find_element_by_id('tabpanel2').text    
    #scoll down so the website will show rating and review box
    no_of_pagedowns = 3
    while no_of_pagedowns:
        elem = driver.find_element_by_tag_name("body")
        elem.send_keys(Keys.PAGE_DOWN)
        time.sleep(1)
        no_of_pagedowns-=1    
    #rating 
    #there may be some product don't have ratings 
    try:
        ra = driver.find_element_by_class_name('css-1eqf5yr').text
        df.rating[i] = float(ra.split(' / ')[0])
    except NoSuchElementException:
        df.rating[i] = np.nan    
    #pic
    driver.find_element_by_class_name('css-1juuxmf').click()
    time.sleep(1)
    df.pic[i] = driver.find_element_by_class_name('css-1glglyy').get_attribute('src')   
    #productsize
    try:
        s = driver.find_element_by_class_name('css-12wl10d').text
        df.productsize[i] = s    
    except NoSuchElementException:
        s = driver.find_element_by_class_name('css-1qf1va5').text
        df.productsize[i] = s
{% endhighlight %}

### (3) User reviews
After completing the scraping for product information, we are going to scrape user ratings and reviews. 

However, by default only 6 reviews are shown in product review box, and each additional click reveals 6 more reviews. Given that some products have thousands of reviews, doing the clicking manually would be too tedious!
<figure>
  <a href="https://miro.medium.com/max/3972/1*0DjcD6zyy6AH1ttCeHistQ.png"><img src="https://miro.medium.com/max/3972/1*0DjcD6zyy6AH1ttCeHistQ.png"></a>
  <figcaption><a>A typical user review</a></figcaption>
</figure>
Being a clever data scientist, I wrote a function to automate the clicking until all reviews were revealed :)
Also, since some reviews may have missing information like headers, we filled in all blank fields with NaN to ensure that each column has same length.

{% highlight python %}
def click(x): 
    try:
        driver.find_element_by_class_name(x).click()
        time.sleep(0.2)
        click(x)
    except ElementNotVisibleException:
        driver.find_element_by_class_name('css-fslzaf').click()
        click(x)
    except NoSuchElementException:
        pass
    
def fillin(rawlist,infolist):
    for i in range(len(infolist)):
        if str(infolist[i]) in str(rawlist[i]):
            pass
        else:
            infolist.insert(i,np.nan)
    while len(infolist) < len(rawlist):
        infolist.append(np.nan) 
{% endhighlight %}

Next, I will use `URL` again to scrap user reviews for all products and save them to a dataframe. For this dataset, it consists features which are:
* `Category`: Three product categories
* `brand`: Product brand 
* `name`: Product name
* `user`: User name
* `stars`: User rating, range from 1 to 5
* `short_review`: User's short review, usually just one sentence
* `long_review`: User's full review
* `helpfulness`: helpfulness of review, voted by other users
* `time`: Time posted

{% highlight python %}
review = pd.DataFrame(columns=['Category','brand', 'name', 'user', 'stars','short_review', 'long_review','helpfulness','time'])
driver = webdriver.Chrome('./chromedriver')
for i in range(len(df)):
    driver.get(df.URL[i])
    time.sleep(4)     
    #create a dataframe
    df_review = pd.DataFrame(columns=['Category','brand', 'name', 'user', 'stars','short_review', 'long_review','helpfulness','time'])
    #scroll page down and let review boxes show
    no_of_pagedowns = 5
    while no_of_pagedowns:
        elem = driver.find_element_by_tag_name("body")
        elem.send_keys(Keys.PAGE_DOWN)
        time.sleep(0.5)
        no_of_pagedowns-=1
    time.sleep(4)
    start_time = time.time()
    click('css-1phfyoj')
    print(time.time()-start_time)
    #user profile
    up = [e.text for e in driver.find_elements_by_class_name('css-81z9n1')][1:-2]   
    #user review
    ur = [e.text for e in driver.find_elements_by_class_name('css-12yc2vd')]    
    #user name
    name =  [e.text for e in driver.find_elements_by_class_name('css-10n46hy')]
    #fill in missing user name with nan
    fillin(up,name)    
    #get stars 
    stars = [int(e.get_attribute('aria-label')[0]) for e in driver.find_elements_by_class_name('css-5quttm')]    
    #short review
    sr = [e.text for e in driver.find_elements_by_class_name('css-ai9pjd')]
    #fill in missing values with nan
    fillin(ur,sr)    
    #long review
    lr = [e.text for e in driver.find_elements_by_class_name('css-1p4f59m')]    
    #helpfulness 
    h = [e.text for e in driver.find_elements_by_class_name('css-39esqn')]
    helpfulness=[int(re.findall('(\d+)',e)[1]) for e in h] 
    #time posted
    t = [e.text for e in driver.find_elements_by_class_name('css-1mfxbmj')]
    
    df_review.user = name
    df_review.stars = stars
    df_review.short_review = sr
    df_review.long_review = lr
    df_review.helpfulness = helpfulness
    df_review.time = t    
    df_review.loc[:,'Category'] = df.Category[i]
    df_review.loc[:,'brand'] = df.brand[i]
    df_review.loc[:,'name'] = df.name[i]    
    review = pd.concat([review,df_review])
{% endhighlight %}



## Data cleaning and pre-processing
Now that we have collected the data, the next step would be cleaning and pre-processing!
### (1) Product information
* drop bundled products, and those with NaN fields,
* remove the “$” sign to convert price to float,
* find products have identical names and reassign them to a new name,
* manually fill in sizes and ingredients for some products and clean them.

In order to compare prices for products better, we create a new feature:
`price_oz`: product price per oz
{% highlight python %}
#based on size and price of product, add a new column price per oz called price_oz
price=[]
for i in index:
    num = df.price[i]/df.oz[i]
    price.append(num)
df = df.assign(price_oz = price)
{% endhighlight %}

### (2) User review
For user reviews, we drop rows that have no user name, missing review field, and duplicate entries. Also, we add a new column for each review:
`id`: Product id, which is the index number for product ininformation dataset
{% highlight python %}
#drop rows that dont have username or reviews
review.dropna(subset=['user', 'long_review'], inplace = True)
#drop deplicated reviews
review.drop_duplicates(subset = ['long_review'],inplace = True)
#assign id number to each review
l = []
n = 0
for i in range(len(review)):
    try:
        if review.name[i] == review.name[i+1]:
            l.append(n)
        else:
            l.append(n)
            n = n+1
    except KeyError:
        l.append(n)
review['id'] = l
{% endhighlight %}

After completing data cleaning, we have a total of **281** products from **three** categories and **137,685** reviews from **110,604** users.

### (3) Repurchase rate
Based on user review dataset, we calculate the **repurchase rate**. Repurchase rate is a strong indication of user satisfaction with the product.

* `Product`: Product name
* `total`: Total number of purchase
* `repurchse`: Number of repurchase orders
* `rate`: Repurchase rate

{% highlight python %}
l = []
l1 = []
l2 = []
l3 = []
for i in range(281):
    frequency = review[review.id == i].user.value_counts()
    total = len(frequency)
    repurchase = sum([1 for i in frequency if i >1])
    rate = repurchase/total
    name = list(review[review.id == i].full_name)[0]   
    l.append(name)
    l1.append(total)
    l2.append(repurchase)
    l3.append(rate)    
df_repurchase = pd.DataFrame({'product': l, 'total':l1,'repurchase': l2, 'rate': l3})
{% endhighlight %}


In this post, I talked about data collection and data cleaning. I will continue rest of parts on how to build a hybrid recommender system for skin care products.



