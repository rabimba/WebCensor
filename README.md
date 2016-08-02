# Web Censorshop Monitor

# requirements

- `Python 3.4`
- `virtualenv 3`
- `pip`

# setup

    virtualenv venv
    source venv/bin/activate
    pip3 install -r requirements.txt

# run

    ./run.py

Then go to `http://127.0.0.1:8080/`

# design

## Read a list of web pages and content from a configuration file

The configuration file `web_monitor.yaml` is parsed at application startup.
it contains the following sample data :

    interval: 10
    sites:
        id1:
            url: 'https://www.facebook.com/'
            content: 'facebook'
            full_match: false

- `interval` : number of seconds between consequitive requests on a website
- `url` : website's url to be tested
- `content` : website's content that should be matched
- `full_match` : boolean which will be used by the regex engine to switch between
    `re.search` (partial match) or `re.match` (full match)

## Periodically make an HTTP request to each site

I call the 'monitor' function to run the flask application in repeat

    sched = BackgroundScheduler()
    sched.add_job(monitor, 'interval', seconds=config['interval'], args=[config])
    sched.start()

The `monitor` function has to check that every website
is available and matches the required content, by calling the
`check_website` function.

It imports `multiprocessing` module to use the `ThreadPool` class,
so that the program can execute multiple checks in parallel.

    pool = ThreadPool(4)
    results = pool.map(check_website, [x[1] for x in config['sites'].items()])

The results are printed on the log output, using `pformat` to prettify them.

    logging.info(pprint.pformat(results))

A `mutex` is used to ensure that when we update the global variable `last_status`,
it won't be read by the Flask view code at the same time.

## Verifies that the page content received from the server matches the content requirements

The following code checks that a webpage content received matches the content
given in the configuration file :

        if self.full_match:
            match_func = re.match
        else:
            match_func = re.search
        if match_func(r'{}'.format(self.content), r.text):
            self.status['match'] = True
        else:
            self.status['match'] = False

## Measures the time it took for the web server to complete the whole request

The following is responsible for measuring the time required to received
the HTTP response :

        start = datetime.datetime.now()
        r = requests.get(self.url, timeout=Site.TIMEOUT)
        # force to download all content
        r.content
        end = datetime.datetime.now()
        self.status['code'] = r.status_code
        self.status['elapsed'] = end - start

## Writes a log file that shows the progress of the periodic checks

The log file is handled with the standard python module `logging`.
Here we configure the logger output on both `stdout` and `web_monitor_turk.log` :

    def init_logger():
        logger = logging.getLogger()
        # log on stdout
        logger.addHandler(logging.StreamHandler())
        # log on LOG_FILE
        file_handler = logging.FileHandler(LOG_FILE)
        logger.addHandler(file_handler)
        logger.setLevel(LOG_LEVEL)

And there we log the new website reported status into the log output, using
`pprint` to have a more readable format :

    # write new entry into log file
    logging.info(pprint.pformat(check))

## Implement a single-page HTTP server interface

Used flasks inbuilt functions

Our architecture is splitted into modules :

    app/
        mod_webmonitor/
            controller.py
            [view.py]
            [model.py]
        static/
        templates/
            webmonitor/
                show.html


## The checking period must be configurable via a command-line option

The module `docopt` has been used to easily define new command line parameters.
Here the `-c=INTERVAL` swicth is defined :

    """
    Usage:
        run.py [options]

    options:
        -c=INTERVAL         Change check interval value
        -h --help           Show this screen.
    """

The configuration is then overwritten after it has been read :

    # overwrite config with cmdline values
    check_interval_cmdline = cmdline['-c']
    if check_interval_cmdline:
        config['check'] = check_interval_cmdline

## The log file must contain the checked URLs, their status and the response times

The following format is printed in the log file :

    {'date': datetime.datetime(2016, 7, 22, 2, 7, 7, 953550),
     'sites': ['code': 200,
  'config_site': {'content': 'What’s Happening',
                  'full_match': False,
                  'url': 'https://www.twitter.com'},
  'elapsed': 0.761493,
  'error': None,
  'match': True,
  'up': True},
]}

- `date` : contains the `datetime` just before we began to check websites availability.
- `sites` : contains the status report for each website
- `code` : corresponding HTTP status code
- `config_site` :  a hash describe the website configuration
- `elasped` the delta between the moment where we started the requested, and the moment when we received the full answer
- `error` : an error describing a problem at application level that might have happened during the test (`SSLErrorè, `TimeoutError`, `ConnectionError`, ...)
- `match` : if the content received and the string describing the content in the configuration file have matched
- `up` : if the website has given a response, and therefore is up
