+++
author = "hannibal"
categories = ["Ruby"]
date = "2016-10-06"
title = "RScrap scraper"
url = "/2016/10/06/rscrap-ruby-scraping-with-cronjob-scripts"

+++

# Intro

Hey folks.

So, there is this project called [Huginn](https://github.com/cantino/huginn) which I absolutely love.

But the thing is, that for a couple of scrappers ( at least for me ), I don't want to spin up a whole rails app.

Hence, I've come up with [RScrap](https://github.com/Skarlso/rscrap). Which is a bunch of Ruby scripts run as cron jobs on a raspberry pi. And because I dislike emails as well, and most of the time, I don't read them, I opted for a nicer solution. Enter the world of [Telegram](https://telegram.org). They provide you with the ability to create bots. You basically get an API key, and than using that key, you can send private messages, or even create an interactive bot which you can send messages too.

In my simple example, I'm using it to send private messages to myself, but I could just as well, make it interactive and than tell it to run one of the scripts.

# The Code

Let's take a look at what we got.

## The main scraper

The main scraper, is simply bunch of convenience methods that wrap handling and working with the database and the telegram bot. That's all. It's very simple. Very short. The Telegram part is just this bit:

~~~ruby
def send_message(text)
  Telegram::Bot::Client.run(@token) do |bot|
    bot.api.send_message(chat_id: @id, text: text)
  end
end
~~~

Straightforward. Creating an interactive bot, would look something like this:

~~~ruby
#!/usr/bin/env ruby
require 'telegram/bot'

token = 'YOUR_TELEGRAM_BOT_API_TOKEN'

Telegram::Bot::Client.run(token) do |bot|
  bot.listen do |message|
    case message.text
    when '/start'
      bot.api.send_message(chat_id: message.chat.id, text: "Hello, #{message.from.first_name}")
    when '/stop'
      bot.api.send_message(chat_id: message.chat.id, text: "Bye, #{message.from.first_name}")
    end
  end
end
~~~

Basically, it will listen, and than you can send it messages and based on the parsed `message.text` you can define functions to call. For example, for rscrap I could define something like `run_script(script)`. And the command would be: `/run reddit`. Which will execute my reddit script. The possibilities are endless.

## The scripts

The scripts use nokogiri to parse a web page, and than return a URL which will be sent by the TelegramBot. They are also saved in the database so that when a new comic strip comes out, I know that it's new. For reddit, I'm saving a timestamp as well, and I collect everything after that timestamp through the reddit API as JSON, and send it as a bundled message with shortified links to the posts using bit.ly.

The scraping is most of the times the same for every comic. Thus, there is a helper method for it. The script itself, is very short. For example, lets look at gunnerkrigg court.

~~~ruby
require_relative '../rscrap'
require 'nokogiri'
require 'open-uri'

url = 'http://www.gunnerkrigg.com'
scrap = Rscrap.new
page = Nokogiri::HTML(open(url))
comic_id = page.css('img.comic_image')[0].select { |e| e if e[0] == 'src' }[0][1]
new_comic = "#{url}#{comic_id}"
scrap.send_new_comic(url, new_comic)
~~~

The interesting part of it is this bit: `comic_id = page.css('img.comic_image')[0].select { |e| e if e[0] == 'src' }[0][1]`. It extracts the URL for the comic image, and stores it as an "id" of the comic. This than, is sent as a message which Telegram will embed. There is no need to visit the web page, the image is in your feed and you can view it directly. Just like an RSS ready.

## Cron

These scripts are best used in a cron job. The comics are usually running with a daily frequency, where as the reddit gatherer is running with an hour frequency. Basically, I'm receiving updates on an hourly basis if there are new posts by then. Running ruby from cron was a bit tricky. I'm using bundler for the environment, and came up with this:

~~~bash
0 6-23 * * * /bin/bash -l -c 'cd /home/<youruser>/rubyproj/rscrap && bundle exec ruby scripts/reddit.rb'
0 8,22 * * * /bin/bash -l -c 'cd /home/<youruser>/rubyproj/rscrap && bundle exec ruby scripts/gunnerkrigg.rb'
0 8,22 * * * /bin/bash -l -c 'cd /home/<youruser>/rubyproj/rscrap && bundle exec ruby scripts/aws_blog.rb'
0 5,23 * * * /bin/bash -l -c 'cd /home/<youruser>/rubyproj/rscrap && bundle exec ruby scripts/goblinscomic.rb'
0 6,20 * * * /bin/bash -l -c 'cd /home/<youruser>/rubyproj/rscrap && bundle exec ruby scripts/xkcd.rb'
0 7,19 * * * /bin/bash -l -c 'cd /home/<youruser>/rubyproj/rscrap && bundle exec ruby scripts/commitstrip.rb'
0 8 * * * /bin/bash -l -c 'cd /home/<youruser>/rubyproj/rscrap && bundle exec ruby scripts/sequiential_art.rb'
~~~

And a telegram message for all these things, looks like this:
Reddit:
![TelegramIMReddit](https://github.com/Skarlso/rscrap/raw/master/shorten.png)
Comics:
![TelegramIMComics](https://github.com/Skarlso/rscrap/raw/master/rscrap2.png)

# Conclusion

That's it folks. Adding a new scraper is easy. I added the aws blog as a new entry as well by just copying the comics scripts. And I'm also getting Weather Reports delivered every morning to me.

Have fun. Any questions, please feel free to leave a comment!

Thanks,
Gergely.
