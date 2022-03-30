# Web Scraping Challenge - Mission to Mars
In this assignment, you will build a web application that scrapes various websites for data related to the Mission to Mars and displays the information in a single HTML page.

-----

# Part 1: Scraping 
Complete initial scraping using Jupyter Notebook, BeautifulSoup, Pandas, and Requests/Splinter.

###### Set up Mission to Mars jupyter notebook
```python
# Import Splinter, BeautifulSoup, and Pandas
from splinter import Browser
from bs4 import BeautifulSoup as soup
import pandas as pd
from webdriver_manager.chrome import ChromeDriverManager

# Set up Splinter
executable_path = {'executable_path': ChromeDriverManager().install()}
browser = Browser('chrome', **executable_path, headless=False)
```

###### NASA Mars News Site
* Scrape the [Mars News Site](https://redplanetscience.com/) and collect the latest News Title and Paragraph Text
* Assign the text to variables that you can reference later

```python
# Visit the Mars news site
url = 'https://redplanetscience.com/'
browser.visit(url)

# Optional delay for loading the page
browser.is_element_present_by_css('div.list_text', wait_time=1)

# Convert the browser html to a soup object
html = browser.html
news_soup = soup(html, 'html.parser')

slide_elem = news_soup.select_one('div.list_text')

slide_elem.find('div', class_='content_title')

# Use the parent element to find the first a tag and save it as `news_title`
news_title = slide_elem.find('div', class_='content_title').get_text()
news_title

# Use the parent element to find the paragraph text
news_p = slide_elem.find('div', class_='article_teaser_body').get_text()
news_p
```

###### JPL Mars Space Images - Featured Image
* Visit the url for the Featured Space Image site [here](https://spaceimages-mars.com/)
* Use splinter to navigate the site and find the image url for the current Featured Mars Image and assign the url string to a variable called featured_image_url
* Make sure to find the image url to the full size .jpg image
* Make sure to save a complete url string for this image

```python
# Visit URL
url = 'https://spaceimages-mars.com'
browser.visit(url)

# Find and click the full image button
full_image_elem = browser.find_by_tag('button')[1]
full_image_elem.click()

# Parse the resulting html with soup
html = browser.html
img_soup = soup(html, 'html.parser')

# find the relative image url
img_url_rel = img_soup.find('img', class_='fancybox-image').get('src')
img_url_rel

# Use the base url to create an absolute url
img_url = f'https://spaceimages-mars.com/{img_url_rel}'
img_url
```

###### Mars Facts
* Visit the [Mars Facts](https://galaxyfacts-mars.com/) webpage and use Pandas to scrape the table containing facts about the planet including Diameter, Mass, etc.
* Use Pandas to convert the data to a HTML table string

```python
# Use `pd.read_html` to pull the data from the Mars-Earth Comparison section
df = pd.read_html('https://galaxyfacts-mars.com')[0]
df.head()

df.columns=['Description', 'Mars', 'Earth']
df.set_index('Description', inplace=True)
df

df.to_html()
```

###### Mars Hemispheres
* Visit the [astrogeology](https://marshemispheres.com/) site to obtain high resolution images for each of Mar's hemispheres
* You will need to click each of the links to the hemispheres in order to find the image url to the full resolution image
* Save both the image url string for the full resolution hemisphere image, and the Hemisphere title containing the hemisphere name
* Use a Python dictionary to store the data using the keys img_url and title
* Append the dictionary with the image url string and the hemisphere title to a list. This list will contain one dictionary for each hemisphere

```python
url = 'https://marshemispheres.com/'
browser.visit(url) 

# Create a list to hold the images and titles.
hemisphere_image_urls = []

# Get a list of all of the hemispheres
links = browser.find_by_css('a.product-item img')

# Next, loop through those links, click the link, find the sample anchor, return the href
for i in range(len(links)):
    hemisphere = {}
    
    # We have to find the elements on each loop to avoid a stale element exception
    browser.find_by_css('a.product-item img')[i].click()
    
    # Next, we find the Sample image anchor tag and extract the href
    sample_elem = browser.links.find_by_text('Sample').first
    hemisphere['img_url'] = sample_elem['href']
    
    # Get Hemisphere title
    hemisphere['title'] = browser.find_by_css('h2.title').text
    
    # Append hemisphere object to list
    hemisphere_image_urls.append(hemisphere)
    
    # Finally, we navigate backwards
    browser.back()

hemisphere_image_urls

browser.quit()
```

# Part 2: MongoDB and Flask Application
Use MongoDB with Flask templating to create a new HTML page that displays all of the information that was scraped from the URLs above.

* Start by converting your Jupyter notebook into a Python script called scrape_mars.py with a function called scrape that will execute all of your scraping code from above and return one Python dictionary containing all of the scraped data
```python
# Import Splinter, BeautifulSoup, and Pandas
from splinter import Browser
from bs4 import BeautifulSoup as soup
import pandas as pd
import datetime as dt
from webdriver_manager.chrome import ChromeDriverManager

def scrape_all():
    # Initiate headless driver for deployment
    executable_path = {'executable_path': ChromeDriverManager().install()}
    browser = Browser('chrome', **executable_path, headless=True)

    news_title, news_paragraph = mars_news(browser)

    # Run all scraping functions and store results in a dictionary
    data = {
        "news_title": news_title,
        "news_paragraph": news_paragraph,
        "featured_image": featured_image(browser),
        "facts": mars_facts(),
        "hemispheres": hemispheres(browser),
        "last_modified": dt.datetime.now()
    }

    # Stop webdriver and return data
    browser.quit()
    return data


def mars_news(browser):

    # Scrape Mars News
    # Visit the mars nasa news site
    url = 'https://redplanetscience.com/'
    browser.visit(url)

    # Optional delay for loading the page
    browser.is_element_present_by_css('div.list_text', wait_time=1)

    # Convert the browser html to a soup object and then quit the browser
    html = browser.html
    news_soup = soup(html, 'html.parser')

    # Add try/except for error handling
    try:
        slide_elem = news_soup.select_one('div.list_text')
        # Use the parent element to find the first 'a' tag and save it as 'news_title'
        news_title = slide_elem.find("div", class_="content_title").get_text()
        # Use the parent element to find the paragraph text
        news_p = slide_elem.find("div", class_="article_teaser_body").get_text()

    except AttributeError:
        return None, None

    return news_title, news_p


def featured_image(browser):
    # Visit URL
    url = 'https://spaceimages-mars.com'
    browser.visit(url)

    # Find and click the full image button
    full_image_elem = browser.find_by_tag('button')[1]
    full_image_elem.click()

    # Parse the resulting html with soup
    html = browser.html
    img_soup = soup(html, 'html.parser')

    # Add try/except for error handling
    try:
        # find the relative image url
        img_url_rel = img_soup.find('img', class_='fancybox-image').get('src')

    except AttributeError:
        return None

    # Use the base url to create an absolute url
    img_url = f'https://spaceimages-mars.com/{img_url_rel}'

    return img_url


def mars_facts():
    # Add try/except for error handling
    try:
        # use 'read_html' to scrape the facts table into a dataframe
        df = pd.read_html('https://galaxyfacts-mars.com')[0]

    except BaseException:
        return None

    # assign columns and set index of dataframe
    df.columns = ['Description', 'Mars', 'Earth']
    df.set_index('Description', inplace=True)

    # Convert dataframe into HTML format, add bootstrap
    return df.to_html(classes="table table-striped")


def hemispheres(browser):
    url = 'https://marshemispheres.com/'

    browser.visit(url + 'index.html')

    # Click the link, find the sample anchor, return the href
    hemisphere_image_urls = []
    for i in range(4):
        # Find the elements on each loop to avoid a stale element exception
        browser.find_by_css("a.product-item img")[i].click()
        hemi_data = scrape_hemisphere(browser.html)
        hemi_data['img_url'] = url + hemi_data['img_url']
        # Append hemisphere object to list
        hemisphere_image_urls.append(hemi_data)
        # Finally, we navigate backwards
        browser.back()

    return hemisphere_image_urls


def scrape_hemisphere(html_text):
    # parse html text
    hemi_soup = soup(html_text, "html.parser")

    # adding try/except for error handling
    try:
        title_elem = hemi_soup.find("h2", class_="title").get_text()
        sample_elem = hemi_soup.find("a", text="Sample").get("href")

    except AttributeError:
        # Image error will return None, for better front-end handling
        title_elem = None
        sample_elem = None

    hemispheres = {
        "title": title_elem,
        "img_url": sample_elem
    }

    return hemispheres


if __name__ == "__main__":

    # If running as script, print scraped data
    print(scrape_all())
```
* Next, create a route called /scrape that will import your scrape_mars.py script and call your scrape function
* Store the return value in Mongo as a Python dictionary
* Create a root route / that will query your Mongo database and pass the mars data into an HTML template to display the data
```python
from flask import Flask, render_template, redirect, url_for
from flask_pymongo import PyMongo
import scrape_mars

app = Flask(__name__)

# Use flask_pymongo to set up mongo connection
app.config["MONGO_URI"] = "mongodb://localhost:27017/mars_app"
mongo = PyMongo(app)

# Or set inline
# mongo = PyMongo(app, uri="mongodb://localhost:27017/mars_app")

@app.route("/")
def index():
    mars = mongo.db.mars.find_one()
    return render_template("index.html", mars=mars)


@app.route("/scrape")
def scrape():
    mars = mongo.db.mars
    mars_data = scrape_mars.scrape_all()
    mars.update({}, mars_data, upsert=True)
    return redirect('/', code=302)


if __name__ == "__main__":
    app.run()
```
* Create a template HTML file called index.html that will take the mars data dictionary and display all of the data in the appropriate HTML elements. Use the following as a guide for what the final product should look like, but feel free to create your own design
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Mission to Mars</title>
    <link
      rel="stylesheet"
      href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css"
    />
  </head>
  <body>
    <div class="container">
      <!-- Add Jumbotron to Header -->
      <div class="jumbotron text-center">
        <h1>Mission to Mars</h1>
        <!-- Add a button to activate scraping script -->
        <p><a class="btn btn-primary btn-lg" href="/scrape" role="button">Scrape New Data</a></p>
      </div>

      <!-- Add section for Mars News -->
      <div class="row" id="mars-news">
        <div class="col-md-12">
          <div class="media">
            <div class="media-body">
              <h2>Latest Mars News</h2>
              <h4 class="media-heading">{{ mars.news_title }}</h4>
              <p>{{ mars.news_paragraph }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Section for Featured Image and Facts table -->
      <div class="row" id="mars-featured-image">
        <div class="col-md-8">
          <h2>Featured Mars Image</h2>

          <!-- if images is False/None/non-existent, then default to error message -->
          <img
            src="{{mars.featured_image | default('static/images/error.png', true) }}"
            class="img-responsive"
            alt="Responsive image"
          />
        </div>

        <div class="col-md-4">
          <!-- Mars Facts -->
          <div class="row" id="mars-facts">
            <h4>Mars Facts</h4>
            {{ mars.facts | safe }}
          </div>
        </div>
      </div>

      <!-- Section for Mars Hemispheres -->
      <div class="row" id="mars-hemispheres">
        <div class="page-header">
          <h2 class="text-center">Mars Hemispheres</h2>
        </div>

        {% for hemisphere in mars.hemispheres %}
        <div class="col-md-3">
          <div class="thumbnail">
            <img src="{{hemisphere.img_url | default('static/images/error.png', true)}}" alt="...">
            <div class="caption">
              <h3>{{hemisphere.title}}</h3>
            </div>
          </div>
        </div>
        {% endfor %}
      </div>
    </div>
  </body>
</html>
```