I thought this would be a fun exercise, there's a lot of reasons you would want to parse apache logs, 
one of the more flashy ones is checking for anomalous behavior. Whatever the case is, you need to parse it first.

So the next question is what log are we going to parse? [here](https://httpd.apache.org/docs/2.4/logs.html) is a list.
The access log is described to "[record] all requests processed by the server." Sounds like a great idea.

Now we actually need some example logs. We could make our own, but where's the fun in that? Let's find some on Google!
Search for `inurl:access.log filetype:log` This is ["Google hacking"](https://en.wikipedia.org/wiki/Google_hacking) if you are not familiar.

I found [this](http://www.almhuette-raith.at/apache-log/access.log), which is 993MB with 5,072,515 entries. The log covers access from December 12th, 2015 to March 23rd, 2020. Which is absolutely perfect for this.

A note about apache - log formats can be changed. We *could* make a generic parser for the default config (found [here](http://httpd.apache.org/docs/2.4/mod/mod_log_config.html#logformat)), but this example uses a custom format. So in the interest of using concrete examples, we'll pick a few fields to parse. If you are curious about the apache log formats (which you should be), the default format is `LogFormat "%h %l %u %t \"%r\" %>s %b"`. The variables here are similar to python's [datetime formatting](https://docs.python.org/2/library/datetime.html) and can be found [here](http://httpd.apache.org/docs/2.4/mod/mod_log_config.html#formats). With that in mind, we're going to manually write some parsing rules and see how it goes!


But first let's take a step back. Since apache will ensure a consistent format, let's just play with the first few lines while writing our parser. That will save us from having to wait for the entire file to be processed more than once. Do that by using [head](https://linux.die.net/man/1/head) `head -n5 access.log > test_access.log`.

So let's start by working with the first line
```python
log_file_line = [line for line in open('test_access.log')][0]
print(log_file_line)
'109.169.248.247 - - [12/Dec/2015:18:25:11 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"\n'
```
So here we see a lot, but let's pick out a few useful things to pull out of this. Here's my list:
1. Remote address
2. Timestamp
3. First line of request
4. Response code
5. Response size
6. User-agent string

So let's slice this into a dictionary. But there's a problem - The first line of the request and the user-agent string are in quotes and have spaces. So we can't just split it like the rest of the columns. *In walks [regular expressions](https://en.wikipedia.org/wiki/Regular_expression).* A regular expression is "a sequence of characters that define a search pattern" - super straight forward. Python has a regular expression package ([re](https://docs.python.org/3.7/library/re.html)) that we will be using today. I highly recomend looking at the [examples](https://docs.python.org/3.7/library/re.html#regular-expression-examples) in the docs.

So let's pull out the fields that don't require the re package first.
```python
log_file_line_dic = {}
log_file_line_dic['remote_addr'] = non_quoted_rows[0]
log_file_line_dic['response_status'] = non_quoted_rows[8]
log_file_line_dic['response_size'] = non_quoted_rows[9]
print(log_file_line_dic)
{'remote_addr': '109.169.248.247',
 'response_status': '200',
 'response_size': '4263'}
```

Now let's add in the regex stuff
```python
log_file_line_dic['remote_addr'] = non_quoted_rows[0]
log_file_line_dic['response_status'] = non_quoted_rows[8]
log_file_line_dic['response_size'] = non_quoted_rows[9]
log_file_line_dic['request_first_line'] = re.findall(r'"(.*?)"', log_file_line)[0]
log_file_line_dic['user_agent_string'] = re.findall(r'"(.*?)"', log_file_line)[2]
print(log_file_line_dic)
{'remote_addr': '109.169.248.247',
 'response_status': '200',
 'response_size': '4263',
 'request_first_line': 'GET /administrator/ HTTP/1.1',
 'user_agent_string': 'Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0'}
```

But as you probably noticed, I left out the timestamp. That was intentional. For our parser, I imagine we'll probably want to be able to change the time format easily. So let's write a helper function for it!

First, let's pull out the date using regex
```python
date_str = re.findall(r'\[.*?\]', log_file_line)[0]
print(date_str)
[12/Dec/2015:18:25:11 +0100]
```
Now there are two concerns you should have here:
1. I made a dangerious assumption - that this regex pattern will only pick up *one* match
2. The braces are still there

Let's address these now. We can garentee that this will *always* only return one match because we figured out that there is only one timestamp "column" in the apache format. The solution to the second concern is much less intuitive - we'll just include the braces in our datetime format like this.
```python
from datetime import datetime
date_str = re.findall(r'\[.*?\]', log_file_line)
datetime.strptime(date_str[0], "[%d/%b/%Y:%H:%M:%S %z]")
datetime.datetime(2015, 12, 12, 18, 25, 11, tzinfo=datetime.timezone(datetime.timedelta(seconds=3600)))
```

But that's super ugly - I don't want to store an object in my return and I assume you don't either. So here's another two birds with one stone situation! Let's add a custom datetime format using datetime's [strftime](https://docs.python.org/3/library/datetime.html#strftime-and-strptime-format-codes) method!
```python
date_str = re.findall(r'\[.*?\]', log_file_line)
datetime.strptime(date_str[0], "[%d/%b/%Y:%H:%M:%S %z]").strftime('%c %z')
'Sat Dec 12 18:25:11 2015 +0100'
```

Now just put it in a simple function, and we're ready to write the parser!
```python
def get_date_from_log_file_line(log_file_line, date_format='%c %z'):
    date_str = re.findall(r'\[.*?\]', log_file_line)
    return datetime.strptime(date_str[0], "[%d/%b/%Y:%H:%M:%S %z]").strftime(date_format)
print(get_date_from_log_file_line(log_file_line))
'Sat Dec 12 18:25:11 2015 +0100'
```

Now for the actual parser! It's been a trip but we still have some major decisions to make:
1. What format do we want our return?
2. What options do we want to specify?

I like [pandas](https://en.wikipedia.org/wiki/Pandas_(software)), so we'll cast a list of dicts to a pandas [DataFrame](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html). But what do we want to specify? I like the idea of keeping it simple - We'll give the parser a file object and *optionally* a date format (which will really help when comparing apache logs aganst other datasets).

So let's write the core section first - parsing each line into a dict and adding it to a list. I want to start with another helper function, parse_line. Don't forget to add some type of check for missing columns (shouldn't ever happen but it could).
```python
def parse_access_log_file_line(log_file_line, date_format='%c %z'):
    log_file_line_dic = {}
    non_quoted_rows = log_file_line.split()
    
    if len(non_quoted_rows) > 1:
        log_file_line_dic['remote_addr'] = non_quoted_rows[0]
        log_file_line_dic['response_status'] = non_quoted_rows[8]
        log_file_line_dic['response_size'] = non_quoted_rows[9]
    log_file_line_dic['request_first_line'] = re.findall(r'"(.*?)"', log_line)[0]
    log_file_line_dic['datetime'] = get_date_from_log_line(log_line, date_format)
    
    return log_file_line_dic
```

Now let's put it all together!
```python
import re
from datetime import datetime

def get_date_from_log_file_line(log_file_line, date_format='%c %z'):
    date_str = re.findall(r'\[.*?\]', log_file_line)
    return datetime.strptime(date_str[0], "[%d/%b/%Y:%H:%M:%S %z]").strftime(date_format)
    
def parse_access_log_file_line(log_file_line, date_format='%c %z'):
    log_file_line_dic = {}
    non_quoted_rows = log_file_line.split()
    
    if len(non_quoted_rows) > 1:
        log_file_line_dic['remote_addr'] = non_quoted_rows[0]
        log_file_line_dic['response_status'] = non_quoted_rows[8]
        log_file_line_dic['response_size'] = non_quoted_rows[9]
    log_file_line_dic['request_first_line'] = re.findall(r'"(.*?)"', log_line)[0]
    log_file_line_dic['datetime'] = get_date_from_log_line(log_line, date_format)
    
    return log_file_line_dic
    
def parse_apache_access_log_file(apache_access_log_file, date_format='%c %z'):
    log = []
    for log_file_line in apache_access_log_file:
        log.append(parse_access_log_file_line(log_file_line, date_format))
    return pd.DataFrame(log)
```

So if we run this with our test data from earlier we get
```python
apache_access_log = parse_apache_access_log_file(open('test_access.log'))

# We also get to use awesome pandas features with our log like the average value per each column
print(log['response_size'].astype('int64').mean())
4365.666666666667
```