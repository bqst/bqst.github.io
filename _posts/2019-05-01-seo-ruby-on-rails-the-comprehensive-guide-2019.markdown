---
layout: post
title: "SEO & Ruby On Rails : the comprehensive guide 2019"
date: 2019-05-01 14:45:10 +0100
categories: [seo, web development]
tags: [ruby on rails, seo]
author: bqst
summary: "This article will focus on the best practices for optimizing your Ruby on Rails web application for search engines (SEO)."
---

🇫🇷 French version available [here](https://www.la-revanche-des-sites.fr/blog/seo-ruby-on-rails-le-guide-complet) !

At [la revanche des sites](https://www.la-revanche-des-sites.fr/), we believe there is a better way to do marketing. A more valuable, less invasive way where customers are earned rather than bought. That’s why we focus on search engine optimization (SEO) to help you to drive customers to your website or directly to your shop.

Many of our client websites are built with **Ruby On Rails**, a **web application framework** designed to work with the Ruby programming language. In this article we regroup the most common techniques we used to optimise your ranking with some examples & tips for your Rails application.

View source code on [Github](https://github.com/larevanchedessites/seo-ruby-on-rails).

## SEO-Friendly URL

Let’s start with the URL. Rails use by default the primary key (also called id) to generate routes and URL for your resources. For example, if you create a new post template, by default, the URL that accesses the resource will be of the type /posts/1 where 1 is the identifier. However, for a **better user experience** (UX) but also to **perform in the search engines**, we generally advise that the URL are composed of keywords associated with the resource. These URL must be **valid and unique**!

You can develop a system for managing this yourself quite easily but for the sake of speed and maintainability, we will use here a gem (or library) called [friendly_id](https://github.com/norman/friendly_id). This allows you to generate the link from a string associated with the resource, also called **slug**, which will not contain special characters (accents, etc.) and **avoid duplication** (two resources can not be accessed via the same URL). In our example, /posts/1 will become /posts/your-title where your-title is a chosen field of the template.

After following the instructions for installing the gem, this gives you:

```ruby
# app/models/post.rb

class Post < ApplicationRecord

  # friendly url
  extend FriendlyId
  friendly_id :title, use: :slugged

end
```

```ruby
# app/controllers/posts_controller.rb

def show
  @post = Post.friendly.find(params[:id])
end
```

## Dynamic Error Pages

The error pages are used to indicate to the user but also to the robots that a resource is not available. Technically, your Rails server will respond with a different HTTP code depending on the state of the resource. **These pages are important for your SEO** since they are used by search engines, including Google, to better understand the behaviour and architecture of your site.

By default, Rails offers pages **404** (page does not exist or more), **422** (incomplete or incorrect resource) and 500 (server error) available in the / public directory. You will need to customize these pages to reflect the overall **design of your site** (with your layout / application.html.erb) or to **dynamically return content based on the user**.

To customize them, you will first need to specify routes to these error pages:

```ruby
# config/routes.rb

match "/404", to: "errors#not_found", via: :all
match "/422", to: "errors#unacceptable", via: :all
match "/500", to: "errors#internal_server_error", via: :all
```

Then create the associated controller:

```ruby
# app/controllers/errors_controller.rb

# do not forget to delete public/{404, 422, 500}.html
# rm public/{404, 422, 500}.html
class ErrorsController < ApplicationController

  def not_found
    render status: 404
  end

  def unacceptable
    render status: 422
  end

  def internal_server_error
    render status: 500
  end

end
```

Do not forget to delete the old pages present in public:

```bash
rm public / {404, 422, 500} .html
```

Finally, specify to your application that errors are now generated dynamically:

```ruby
# config/application.rb

config.exceptions_app = self.routes
```

## Redirections

After putting your application into production, you may be adding, modifying or deleting some pages of your site or changing the name of one of your posts and thus its **slug**. The architecture of your site will be modified, as well as your URLs. However, if a search engine has already indexed your site, you will **have to redirect your old pages**, and therefore old URLs, to new ones so as not to lose your visitors on your 404 pages. In this case, you must create 301 redirects to specify that one page has been moved permanently to another.

The Rails routing system makes these **301 redirects** quite easily in the routes.rb file:

```ruby
# 301 redirect from old URLs
match "/old_path_to_posts/:id", to: redirect("/posts/%{id}s")
```

You can also define regular expressions (or regexp) to automate some type of redirects.

## Meta Tags

Meta Tags allow search engines and social networks **to better understand the content of your page** and thus gain visibility, so they are essential in SEO. They must be dynamic since they are specific to each page and must be unique to avoid any duplicate content conflicts.

Ruby On Rails does not propose a default management system for Meta Tags, so we will put one in place.

Start by creating a meta.yml file in the config directory that will contain the default values:

```yaml
# config/meta.yml

meta_title: "SEO & Ruby On Rails"
meta_description: "The 2018 comprehensive guide on SEO in Rails"
meta_image: "logo.png" # Une image dans votre dossier app/assets/images/
twitter_account: "@RevancheSites"
```

We will then need to initialize the DEFAULT_META Ruby constant to be able to use the values anywhere in your application:

```ruby
# config/initializers/default_meta.rb

# Initialize default meta tags.
DEFAULT_META = YAML.load_file(Rails.root.join("config/meta.yml"))
```

Then create the methods that will retrieve these values:

```ruby
# app/helpers/meta_tags_helper.rb

module MetaTagsHelper
  def meta_title
    content_for?(:meta_title) ? content_for(:meta_title) : DEFAULT_META["meta_title"]
  end

  def meta_description
    content_for?(:meta_description) ? content_for(:meta_description) : DEFAULT_META["meta_description"]
  end

  def meta_image
    meta_image = (content_for?(:meta_image) ? content_for(:meta_image) : DEFAULT_META["meta_image"])
    # ajoutez la ligne ci-dessous pour que le helper fonctionne indifféremment
    # avec une image dans vos assets ou une url absolue
    meta_image.starts_with?("http") ? meta_image : image_url(meta_image)
  end
end
```

Now add Meta Tags to your layout:

```ruby
/ application.html.slim
title = meta_title

meta name="description" content="#{meta_description}"

/ Facebook Open Graph data
meta property="og:title" content="#{meta_title}"
meta property="og:type" content="website"
meta property="og:url" content="#{request.original_url}"
meta property="og:image" content="#{meta_image}"
meta property="og:description" content="#{meta_description}"
meta property="og:site_name" content="#{meta_title}"

/ Twitter Card data
meta name="twitter:card" content="summary_large_image"
meta name="twitter:site" content="#{DEFAULT_META["twitter_account"]}"
meta name="twitter:title" content="#{meta_title}"
meta name="twitter:description" content="#{meta_description}"
meta name="twitter:creator" content="#{DEFAULT_META["twitter_account"]}"
meta name="twitter:image:src" content="#{meta_image}"
```

And that’s it, you can now simply dynamically set these variables according to your resource, for example here for posts#show:

```ruby
# app/views/posts/show.html.slim

- content_for :meta_title, "#{@post.title} | #{DEFAULT_META["meta_title"]}"
```

## Structured Data

**Structured data helps send the right signals to search engines** about your **app** and **content**, helps to increase search engine understanding of your site’s content, and improve visibility through rich snippets and results. Also increases probability of being selected in **Knowledge Graph**. You really have to make the effort to implement a markup scheme on your site because this data is increasingly important for Google.

To know which schema is used on your site, we recommend [this series of articles from Google](https://developers.google.com/search/docs/guides/intro-structured-data).

There are different formats to use them: **JSON-LD**, **Microdata** or **RDFA**. It is advisable to use JSON-LD by Google but for the sake of simplicity, we will generally use Microdata.

The [green_monkey](https://github.com/Paxa/green_monkey) gem makes it easy to integrate Microdata properties directly into your application:

```ruby
# app/models/post.rb

class Post < ActiveRecord::Base
  html_schema_type :BlogPosting
end

Post.html_schema_type # => Mida::SchemaOrg::BlogPosting
Post.new.html_schema_type # => Mida::SchemaOrg::BlogPosting
```

```ruby
/ show.html.haml
%article[post]
  = link_to "/posts/#{post.id}", :itemprop => "url" do
    %h3[:name]>= post.title
  .post_body[:articleBody]= post.body.html_safe
  = time_tag(post.created_at, :itemprop => "datePublished")
```

## Breadcrumb

The breadcrumb allows you to **remember the location of a page on your site** in the global tree. It is important in SEO since [Google officially recommends to offer a breadcrumb on your website](https://developers.google.com/search/docs/guides/enhance-site?visit_id=1-636307315124148974-4102218133&hl=en&rd=1#enable-breadcrumbs).

There exists some gems to automatically generate a breadcrumb on your application : [gretel](https://github.com/lassebunk/gretel), [loaf](https://github.com/piotrmurach/loaf) or [breadcrumbs_on_rails](https://github.com/weppos/breadcrumbs_on_rails). We will use the latter, easily customizable and **compatible with [Bootstrap](https://getbootstrap.com/)**.

To install it, simply add the gem to your Gemfile.

```ruby
# Gemfile
gem "breadcrumbs_on_rails"
```

And then add to the controllers concerned by the breadcrumb display:

```ruby
# app/controllers/posts_controller.rb

class PostsController < ApplicationController

  add_breadcrumb "Blog", :posts_path

  def index
    @posts = Post.all
  end

  def show
    @post = Post.friendly.find(params[:id])
    add_breadcrumb @post, post_path(@post)
  end

end
```

NB : If you want to add the structured data features on your breadcrumb, you define a constructor :

```ruby
# lib/breadcrumbs/builders/structured_data_breadcrumbs_builder.rb

class Breadcrumbs::Builders::StructuredDataBreadcrumbsBuilder < BreadcrumbsOnRails::Breadcrumbs::Builder
  def render
    @elements.collect do |element|
      render_element(element)
    end.join(@options[:separator] || " &raquo; ")
  end

  def render_element(element)
    if element.path == nil
      content = compute_name(element)
    else
      content = @context.link_to_unless_current(compute_name(element), compute_path(element), element.options.merge({
        itemscope: "",
        itemtype: "http://schema.org/Thing",
        itemprop: "item"
      }))
    end
    if @options[:tag]
      @context.content_tag(
        @options[:tag],
        content,
        {itemprop:"itemListElement", itemscope: "", itemtype:"http://schema.org/ListItem"}
      )
    else
      ERB::Util.h(content)
    end
  end
end
```

Then display your breadcrumb in your layout :

```ruby
/ application.html.slim
ul.breadcrumb itemscope="" itemtype="http://schema.org/BreadcrumbList"
  = render_breadcrumbs tag: 'li', separator: '',
builder: Breadcrumbs::Builders::StructuredDataBreadcrumbsBuilder
```

## Sitemap

The sitemap is the file that contains the **list of your website URL**. This file is in XML format. It gives search engines information about your URL and the architecture of your site. It allows sites with a substantial content and a complex mesh to be better understood by search engines and thus be better referenced.

Some gems can generate it as [sitemap_generator](https://github.com/kjvarga/sitemap_generator), or [dynamic_sitemaps](https://github.com/lassebunk/dynamic_sitemaps) but it is nevertheless possible to build it by yourself thanks to the **XML builder** library.

First, create the route to specify the controller that will handle the request:

```ruby
# config/routes.rb

get '/sitemap.xml' => 'sitemaps#index', defaults: { format: 'xml' }
```

Then let’s take care of building the associated controller:

```ruby
# app/controllers/sitemaps_controller.rb

class SitemapsController < ApplicationController

  layout :false
  before_action :init_sitemap

  def index
    @posts = Post.all
  end

  private

  def init_sitemap
    headers['Content-Type'] = 'application/xml'
  end

end
```

And the associated view :

```ruby
# app/views/sitemaps/index.xml.builder

xml.instruct! :xml, version: '1.0'
xml.tag! 'sitemapindex', 'xmlns' => "http://www.sitemaps.org/schemas/sitemap/0.9" do

  xml.tag! 'url' do
    xml.tag! 'loc', root_url
  end

  xml.tag! 'url' do
    xml.tag! 'loc', contact_url
  end

  @posts.each do |post|
    xml.tag! 'url' do
      xml.tag! 'loc', post_url(post)
      xml.lastmod post.updated_at.strftime("%F")
    end
  end

end
```

## Robots.txt

The robots.txt file is a text file used for the natural referencing of your site, containing **instructions for search engine indexing robots** in order to specify **which pages may or may not be indexed**. You can visit [robots-txt.com](http://robots-txt.com/) for more information about its contents.

Ruby On Rails proposes by default a robots.txt present in the public directory. You can, as with the management of the error pages or the sitemap, make it **dynamic** and allow to **define directives specific to your environment**, development, pre-production or production for example.

```ruby
# config/routes.rb

get "/robots.:format", to: "pages#robots"
```

```ruby
# app/controllers/pages_controller.rb

class PagesController < ApplicationController
  
  def robots
    # Don't forget to delete /public/robots.txt
    respond_to :text
  end

end
```

Also specify the URL of your sitemap to help robots find it easier.

```ruby
/ robots.txt.slim
- if Rails.env == "production"
  = "User-Agent: *\n"
  = "Disallow: \n"
- else
  = "User-Agent: *\n"
  = "Noindex: /\n"
  
= "\nSitemap: #{root_url}sitemap.xml"
```

## AMP

Google AMP, for “**Accelerated Mobile Pages**” is a protocol designed by Google to enable a “faster web”. AMP pages are a variation of classic web pages with simplified HTML and JavaScript. **AMP only keeps what is needed** to display the information very quickly.

First, specify the new mime type for your application :

```ruby
# config/initializers/mime_types.rb

# Be sure to restart your server when you modify this file.

# Add new mime types for use in respond_to blocks:
# Mime::Type.register "text/richtext", :rtf

Mime::Type.register 'text/html', :amp
```

You will then need to create a new AMP layout:

```ruby
<!-- application.amp.erb -->
<!doctype html>
<html ⚡>
  <head>
    <meta charset="utf-8">
    <link rel="canonical" href="<%= url_for(format: :html, only_path: false) %>" >
    <meta name="viewport" content="width=device-width,minimum-scale=1,initial-scale=1">
    <style amp-boilerplate>body{-webkit-animation:-amp-start 8s steps(1,end) 0s 1 normal both;-moz-animation:-amp-start 8s steps(1,end) 0s 1 normal both;-ms-animation:-amp-start 8s steps(1,end) 0s 1 normal both;animation:-amp-start 8s steps(1,end) 0s 1 normal both}@-webkit-keyframes -amp-start{from{visibility:hidden}to{visibility:visible}}@-moz-keyframes -amp-start{from{visibility:hidden}to{visibility:visible}}@-ms-keyframes -amp-start{from{visibility:hidden}to{visibility:visible}}@-o-keyframes -amp-start{from{visibility:hidden}to{visibility:visible}}@keyframes -amp-start{from{visibility:hidden}to{visibility:visible}}</style><noscript><style amp-boilerplate>body{-webkit-animation:none;-moz-animation:none;-ms-animation:none;animation:none}</style></noscript>
    <script async src="https://cdn.ampproject.org/v0.js"></script>
    <script async custom-element="amp-iframe" src="https://cdn.ampproject.org/v0/amp-iframe-0.1.js"></script>
    <script async custom-element="amp-youtube" src="https://cdn.ampproject.org/v0/amp-youtube-0.1.js"></script>
    <% if Rails.application.assets && Rails.application.assets['amp/application'] %>
      <style amp-custom><%= Rails.application.assets['amp/application'].to_s.html_safe %></style>
    <% else %>
      <style amp-custom><%= File.read "#{Rails.root}/public#{stylesheet_path('amp/application', host: nil)}" %></style>
    <% end %>
  </head>
  <body>
    <div class="amp">
      <%= yield %>
    </div>
  </body>
</html>
```

The AMP needs to include only CSS tags and not external tags. The trick to continue using the Rails asset pipeline and compiled CSS in views is to create a new SASS style file under app/assets/stylesheets/amp/application.scss containing the style you want for your pages AMP. Then save this file to the precompilation pipe by adding it to the config/application.rb file.

```ruby
# config/application.rb

config.assets.precompile << 'amp/application.scss'
```

The last problem handled is that AMP limits the use of certain tags and requires modifications for others to work. For example, the img tag becomes amp-img, the same for iframes. The solution is to implement a scrubber to customize the Rails ActionView sanitize method.
This scrubber will try to convert the original DOM to make it “AMP valid”.
That’s what it looks like:

```ruby
# app/scrubbers/amp_scrubber.rb

class AmpScrubber < Rails::Html::PermitScrubber
  TAG_MAPPINGS = {
    'img' => lambda { |node|
      if node['width'] && node['height']
        node.name = 'amp-img'
        node['layout'] = 'responsive'
        node['srcset'] = node['src']
      else
        node.remove
      end
    },
    'iframe' => lambda { |node|
      find_parent(node).add_child(node)

      node['src'] = node['src'].gsub(%r{^(\/\/|http:\/\/)}, 'https://')
      url = URI(node['src'])
      node['layout'] = 'responsive'

      if url.host.include?('youtube.com')
        node.name = 'amp-youtube'
        node['data-videoid'] = node['src'].match(%r{(\/embed\/|watch?v=)(.*)})[2]
        node.remove_attribute('src')
      else
        node.name = 'amp-iframe'
      end
    }
  }.freeze

  def initialize
    super
    @tags = %w(a em p span h1 h2 h3 h4 h5 h6 div strong s u br blockquote)
    @attributes = %w(style contenteditable frameborder allowfullscreen)
  end

  def self.find_parent(node)
    node = node.parent while node.parent
    node
  end

  protected

  def scrub_attribute?(name)
    !super
  end

  def scrub_node(node)
    if node.name.in?(TAG_MAPPINGS.keys)
      remap_node! node, TAG_MAPPINGS[node.name]
    else
      super
    end
  end

  def remap_node!(node, filter)
    case filter
    when String
      node.name = filter
    when Proc
      filter.call(node)
    end
  end
end
```

Finally, you can create the views you want to make available in AMP for example:

```ruby
# app/views/posts/show.amp.slim

h1 = @post.title
div = sanitize @post.content, scrubber: AmpScrubber.new
```

Do not forget to specify the URLs of your AMP pages in your layout to allow Google to index them:

```ruby
# app/views/layouts/application.html.erb
<link rel="canonical" href="<%= url_for(format: :html, only_path: false) %>" >
<link rel="amphtml" href="<%= url_for(format: :amp, only_path: false) %>" >
```

## HTTPS

By default, if you have configured your server correctly with your **SSL certificate**, you should automatically have your Ruby On Rails application in HTTPS. To enable the feature for a specific environment, you will need to enable the following option:

```ruby
# config/environments/production.rb 

Rails.application.configure do 
  # ... 
  # force HTTPS on production 
  config.force_ssl = true 
end
```

## Bonus

### www to non-www redirection

In SEO, one of the recurring problems is the **duplicate content**, which corresponds to two different pages containing the same content. This problem is very often caused by the duplication of content between the non-www (http://example.com) and the www version (http://www.example.com).

To work around this problem, a simple technique:

```ruby
# Force www redirect
# Start server with rails s -p 3000 -b lvh.me
# Then go to http://www.lvh.me:3000
constraints(host: /^(?!www\.)/i) do
  match '(*any)' => redirect { |params, request|
    URI.parse(request.url).tap { |uri| uri.host = "www.#{uri.host}" }.to_s
  }
end
```

### Canonical

The other way to avoid duplicate content is to use the rel = canonical link tag. It allows to indicate to the search engines the canonical URL on a given page.

If any of your site’s pages are accessible through multiple URLs, or if different pages on your site have similar content (for example, a page with a mobile version and a classic version), you must explicitly tell Google what is the canonical URL for these pages, in other words the authoritative URL, otherwise Google will choose the canonical page for you or consider all similar pages equal.

The trick to set it up on your Rails application:

```ruby
/ application.html.slim
= yield :canonical
```

```ruby
# app/helpers/application_helper.rb

def canonical(url)
  content_for(:canonical, tag(:link, rel: :canonical, href: url)) if url
end
```

```ruby
/ show.html.slim
- canonical(blog_post_url(@blog_post))
```

### Cache

We explained in a previous [article on the implementation strategies for SEO](https://www.la-revanche-des-sites.fr/blog/strategie-de-mise-en-cache-pour-le-seo), the cache is not negligible especially if your site contains a lot of URLS. For each crawl of a site, Google’s robot dedicates a specific time. **The faster your pages load, the more Google crawls pages on your site**.

Rails provides a set of caching features. We could devote a whole article to this subject, so I advise you to go to the [official documentation](http://guides.rubyonrails.org/caching_with_rails.html$).

### Compression

When you put a site online, you usually test its technical status with tools like [PageSpeed](https://developers.google.com/speed/) that allows you to know **the rating that Google assigns to your site** in technical terms. It is said that rating awarded by Google PageSpeed, is a **ranking factor for your SEO**, therefore you must aim for the best rating.

Generally, the problems that PageSpeed will raise will be the position of your JavaScript resources as well as the compression of your assets. For the first problem, all you have to do is move calls from them to your layout. For compression, nothing more simple, just activate it with the middleware [Rack :: Deflater](http://rubydoc.info/github/rack/rack/master/Rack/Deflater). Inserted correctly in your application, it can significantly reduce the size of your responses.

```ruby
# config/application.rb

module SeoRubyOnRails
  class Application < Rails::Application
    # Deflater
    # See also : https://robots.thoughtbot.com/content-compression-with-rack-deflater
    config.middleware.use Rack::Deflater
  end
end
```

### Antispam

You have already understood, **in SEO, links (or anchors) are essential**, they allow your visitors and robots to navigate your site. It is really important to know the **nofollow directive**. By default, if a bot looks at a link, it follows it to try to**index the entire page recursively**. This applies to the internal links of your site but also external. It will be necessary to pay attention to all the fields likely to be used for spammers to come to improve the visibility of their own sites by adding links on yours. For this we can use the [Nokogiri](https://github.com/sparklemotion/nokogiri) gem. Just inspect the sent fields that can contain links before saving them.

```ruby
# app/models/post.rb

class Post < ApplicationRecord
  # ...
  
  # callbacks
  before_save :anti_spam

  def anti_spam
    doc = Nokogiri::HTML::DocumentFragment.parse(self.content)
    doc.css('a').each do |a|
      a[:rel] = 'nofollow'
      a[:target] = '_blank'
    end
    self.content = doc.to_s
  end
  # ...
  
end
```

The Nokogiri gem can also be used to create a crawler.

### Homepage H1

In SEO, **a page corresponds to a positioning goal**. We usually include the company logo in the `<h1>` tag for the home page only (which uses the brand name as the alt attribute). All other pages on the site will have a relevant `<h1>` text tag on the page.

```ruby

/ app/views/layouts/_header.html.slim
header
  nav
    ul
      li
        = link_to root_path do
          - if current_page?(root_url)
            h1 = image_tag 'lrds_logo.svg', alt: "SEO & Ruby On Rails", height: "20px"
          - else
            = image_tag 'lrds_logo.svg', alt: "SEO & Ruby On Rails", height: "20px"

      li = link_to "Blog", posts_path
      li = link_to "Contact", contact_path
```

To conclude, Ruby On Rails being a very flexible fast development framework, it is quite easy to set up all kinds of recommendations for SEO. There are many factors to be considered when positioning yourself on search engines, this guide lists only those related directly to development, but we must not forget that original, quality and organized content with a architecture that meets the needs and expectations of your visitors remain the keys to reach the first position!

Feel free to share and ask any questions or remarks in comment, we will be happy to answer!

## Special thanks

[https://www.antoine-brisset.com/blog/seo-ruby-on-rails-1/]
[https://nebulab.it/blog/abc-of-seo-for-ruby-on-rails-developers/]
[https://2017doneright.com/comprehensive-guide-on-seo-in-rails-8b124ca81d37]
[https://www.amberbit.com/blog/2015/4/23/seo-basics-for-rails-developers/]
[https://www.inboundio.com/Blog/seo-for-ruby-on-rails-complete-guide]
[http://codkal.com/seo-ruby-rails-guide/]
[https://www.udemy.com/ruby-on-rails-seo/]
[https://www.lewagon.com/blog/tuto-setup-metatags-rails]
[https://www.pluralsight.com/guides/ruby-ruby-on-rails/using-https-with-ruby-on-rails]
[https://github.com/hardhatdigital/google-page-speed-guidelines]
