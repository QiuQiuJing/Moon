---
layout: post
title: "Recommender System: Skin Care Products (Part 1)"
date: 2019-08-21
excerpt: "Web scraping and data cleaning"
tags: [web scraping, data cleaning, introduction]
comments: false
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



## Web scraping and data collection
All data are scraped from [sephora.com](). There are two parts of information are needed to build recommender system:

* **Product basic information**: categories, brands, product names, ingredients, prices, sizes, ratings, descriptions and pictures
* **User reviews**: user names, ratings, reviews, time and helpfulness

I use [**Selenium with python**](https://selenium-python.readthedocs.io/), because web pages are not static and need to interact with them.

### (1) Urls for products
In this post, I choose three categories for skin care products, which are facial cleanser, toners and moisturisers. First, in each category, 
I scarp links for all selected products and create a dataframe with two features `category` and `url`.

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
df_info = pd.DataFrame(columns=['brand', 'name', 'rating', 'price',
                                'descriptions', 'ingredients','pic','productsize'])
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









{% highlight html %}
{% raw %}
<nav class="pagination" role="navigation">
    {% if page.previous %}
        <a href="{{ site.url }}{{ page.previous.url }}" class="btn" title="{{ page.previous.title }}">Previous article</a>
    {% endif %}
    {% if page.next %}
        <a href="{{ site.url }}{{ page.next.url }}" class="btn" title="{{ page.next.title }}">Next article</a>
    {% endif %}
</nav><!-- /.pagination -->
{% endraw %}
{% endhighlight %}

{% highlight ruby %}
module Jekyll
  class TagIndex < Page
    def initialize(site, base, dir, tag)
      @site = site
      @base = base
      @dir = dir
      @name = 'index.html'
      self.process(@name)
      self.read_yaml(File.join(base, '_layouts'), 'tag_index.html')
      self.data['tag'] = tag
      tag_title_prefix = site.config['tag_title_prefix'] || 'Tagged: '
      tag_title_suffix = site.config['tag_title_suffix'] || '&#8211;'
      self.data['title'] = "#{tag_title_prefix}#{tag}"
      self.data['description'] = "An archive of posts tagged #{tag}."
    end
  end
end
{% endhighlight %}


### Standard Code Block

    {% raw %}
    <nav class="pagination" role="navigation">
        {% if page.previous %}
            <a href="{{ site.url }}{{ page.previous.url }}" class="btn" title="{{ page.previous.title }}">Previous article</a>
        {% endif %}
        {% if page.next %}
            <a href="{{ site.url }}{{ page.next.url }}" class="btn" title="{{ page.next.title }}">Next article</a>
        {% endif %}
    </nav><!-- /.pagination -->
    {% endraw %}


### Fenced Code Blocks

To modify styling and highlight colors edit `/assets/css/syntax.css`. Line numbers and a few other things can be modified in `_config.yml`. Consult [Jekyll's documentation](http://jekyllrb.com/docs/configuration/) for more information.

~~~ css
#container {
    float: left;
    margin: 0 -240px 0 0;
    width: 100%;
}
~~~

~~~ html
{% raw %}<nav class="pagination" role="navigation">
    {% if page.previous %}
        <a href="{{ site.url }}{{ page.previous.url }}" class="btn" title="{{ page.previous.title }}">Previous article</a>
    {% endif %}
    {% if page.next %}
        <a href="{{ site.url }}{{ page.next.url }}" class="btn" title="{{ page.next.title }}">Next article</a>
    {% endif %}
</nav><!-- /.pagination -->{% endraw %}
~~~

~~~ ruby
module Jekyll
  class TagIndex < Page
    def initialize(site, base, dir, tag)
      @site = site
      @base = base
      @dir = dir
      @name = 'index.html'
      self.process(@name)
      self.read_yaml(File.join(base, '_layouts'), 'tag_index.html')
      self.data['tag'] = tag
      tag_title_prefix = site.config['tag_title_prefix'] || 'Tagged: '
      tag_title_suffix = site.config['tag_title_suffix'] || '&#8211;'
      self.data['title'] = "#{tag_title_prefix}#{tag}"
      self.data['description'] = "An archive of posts tagged #{tag}."
    end
  end
end
~~~

### GitHub Gist Embed

An example of a Gist embed below.

{% gist mmistakes/6589546 %}
