---
layout: post
title: "Recommender System: Skin Care Products (Part 1)"
date: 2019-08-21
excerpt: "Web scraping and data cleaning"
tags: [web scraping, data cleaning, introduction]
comments: false
---

### Introduction
A basic skin care routine involves applying products such as facial cleansers, toners and moisturisers. However, many users face problems choosing products that suit them. 
In this post, I will bring you through how to build a recommender system based on:
1. _content-based filtering method_
2. _collaborative filtering method_

Using data from sephora.com, the system recommends a skin care product based on products that a user has liked, or what category of product the user is presently searching for.



### Highlighted Code Blocks

To modify styling and highlight colors edit `/assets/css/syntax.css`.

{% highlight python %}
#step 1 
#Get url for product in each category
driver = webdriver.Chrome('./chromedriver')

#Categories include facial cleanser, toner and moisturizer
productcat = ['face-wash-facial-cleanser', 'facial-toner-skin-toner', 'moisturizer-skincare']

#create a dataframe
df = pd.DataFrame(columns=['Category', 'URL'])

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
