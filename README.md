# User Guide

The API documentation can be read from [here](./APIDocumentation.md)


## 1. USPTO

There are 3 formats, for 3 time periods, and sources, respectively, from https://bulkdata.uspto.gov/:

1. 1976 - 2000:`USPTO1`
2. 2001 - 2004: `USPTO2`
3. 2005 - Current: `USPTO3`

### 1.1. Crawler

[USPTOCrawler](./web_crawler/uspto/uspto_lxml_crawler.py) downloads and extracts the compressed dataset files into folder `uspto_datasets`.

The default directory to store downloaded datasets can be changed by modifying `DEFAULT_OUTPUT_DIRECTORY_FOR_DOWNLOADED_FILES_USPTO` in [web_crawler_constansts.py](./web_crawler/web_crawler_constants.py).

#### Usage

It is recommended to execute `USPTOCrawler.download_datasets(URL)` for each year separatedly, such as https://bulkdata.uspto.gov/data/patent/grant/redbook/fulltext/1993 instead of https://bulkdata.uspto.gov/, since the downloading time is extremely long and there is a high chance of broken files.

However, take note of year 2000, as it contains the files for both version1 and 2, which can result in duplicated patents.

```
from web_crawler.uspto.uspto_lxml_crawler import USPTOCrawler


if __name__ == '__main__':
    urls = [
        'https://bulkdata.uspto.gov/data/patent/grant/redbook/fulltext/2006'
    ]

    crawler = USPTOCrawler(remove_compressed=True)
    downloaded_files = crawler.download_datasets(urls)
```


### 1.2. Parser
[USPTO parser](./dataset_parser/uspto/uspto_parser.py) parses a text file and returns a `Dict` (JSON) object containing only the required information.

The parsers for each version of USPTO, respectively:

1. 1976 - 2000: [uspto_parser_for_version1.py](./dataset_parser/uspto/version1_1976_2001/uspto_parser_for_version1.py)
2. 2001 - 2004: [uspto_parser_for_version2.py](./dataset_parser/uspto/version2_2001_2004/uspto_parser_for_version2.py)
3. 2005 - Current: [uspto_parser_for_version3.py](./dataset_parser/uspto/version3_2005_current/uspto_parser_for_version3.py)

For the ease of usage, [USPTO parser](./dataset_parser/uspto/uspto_parser.py) can auto-identify the version of the input file and parse it.

#### Usage

The schemas for 3 versions are in [require_keywords.py](./dataset_parser/uspto/require_keywords.py).

After filtering the unused attributes, the remains will be renamed and put into the corresponding keys. These refining schema mappings are in [paths_for_dict_to_document_based_schemas.py](./dataset_parser/uspto/paths_for_dict_to_document_based_schemas.py).

```
from dataset_parser.uspto.uspto_parser import USPTOParser


if __name__ == "__main__":
    ## version 1
    #filename = "pftaps19870106_wk01.txt"

    ## version 2
    #filename = "pg021008.XML"

    ## version 3
    filename = "ipg061205.xml"

    # USPTOParser can parser all versions
    parser = USPTOParser()
    output = parser.parse(filename)
```

### 1.3. Crawler + Parser + MongoDBClient

#### Usage

Upon inserting the parsed patents into the database, it is the user's responsibility to ensure that the `SOURCE` keyword (which is `USPTO1`, `USPTO2`, or `USPTO3`) matches the version of the patents.

[uspto_main.py](./uspto_main.py)

```
from dataset_parser.uspto.uspto_parser import USPTOParser
from db_clients.mongodb_client import MongoDBClient
from web_crawler.uspto.uspto_lxml_crawler import USPTOCrawler


if __name__ == '__main__':
    urls = [
        'https://bulkdata.uspto.gov/data/patent/grant/redbook/fulltext/1980',
        'https://bulkdata.uspto.gov/data/patent/grant/redbook/fulltext/2006',
        'https://bulkdata.uspto.gov/data/patent/grant/redbook/fulltext/2003'
    ]

    crawler = USPTOCrawler(remove_compressed=True)
    parser = USPTOParser()

    ## Init a MongoDB client to insert the patents into the database
    ## Default: database='innosenze_development', host='localhost', port=27017
    client = MongoDBClient()
    #client.remove()

    for url in urls:
        downloaded_files = crawler.download_datasets([url])

        for filename in downloaded_files:
            output = parser.parse(filename)

            ## If inserting directly to the database, comment this line
            #parser.write_to_json_file(output, filename, pretty=True)

            ## If inserting directly to the database, uncomment this line
            #client.insert_many(output, 'USPTO1')
```

___

## 2. WIPO
There are 2 formats and sources, respectively:  

1. 1978 - 2017: `WIPO_BEFORE_2018`
2. 2018: `WIPO_2018`

### 2.1. Crawler

Upon building this crawler, WIPO datasets for 2017 and before are shared on `OneDrive`; therefore, only 2018 datasets need to be downloaded using a crawler.

This crawler only downloads 2018 WIPO datasets from a FTP server. The hostname and credentials for the server can be found at [.env](./web_crawler/wipo/.env).

[WIPOCrawler](./web_crawler/wipo/wipo_crawler.py) downloads and extracts the compressed dataset files into folder `wipo_datasets`.

The default directory to store downloaded datasets can be changed by modifying `DEFAULT_OUTPUT_DIRECTORY_FOR_DOWNLOADED_FILES_WIPO` in [web_crawler_constansts.py](./web_crawler/web_crawler_constants.py).

#### Usage

[wipo_main.py](./wipo_main.py)

```
crawler = WIPOCrawler()

# Download all dataset files to folder `wipo_datasets`
crawler.download_datasets()

# Extract the datasets
datasets = crawler.extract_compressed(remove_compressed=True)

crawler.close()
```

### 2.2. Parser

[WIPO parser](./dataset_parser/wipo/wipo_parser.py) parses a text file and returns a `Dict` (JSON) object containing only the required information.

Unlike `USPTO` where there are 2 different file types, `WIPO` datasets are only different in their schemas.

#### Usage

The schemas and refining schema mappings for both versions are in [schema.py](./dataset_parser/wipo/schema.py).

```
filename = '50.xml'
parser = WIPOParser()
output = parser.from_file(filename)
```

### 2.3. Crawler + Parser + MongoDBClient

#### Usage

Upon inserting the parsed patents into the database, it is the user's responsibility to ensure that the `SOURCE` keyword (which is `WIPO_BEFORE_2018` or `WIPO_2018`) matches the version of the patents.

The patents in WIPO are separated into 1 patent for each `.xml` file. Storing all patents for 1 year in an array causes a HUGE memory usage. Therefore, it is recommended to set a limit number to release the patents in the array.

Here are the helper functions for both WIPO 2018 and 2017-before:
```
import json
import os.path


def create_the_output_directory(output_directory):
    """Create the output directory if it doesnt not exist."""

    _current_directory = get_current_directory()
    path = join_os_path(_current_directory, output_directory)
    if file_doesnt_exist(path):
        os.makedirs(path)

def get_current_directory():
    return os.getcwd()

def join_os_path(*args):
    return os.path.join(*args)

def file_doesnt_exist(path):
    return not os.path.exists(path)

def get_all_files_in_directory(root=''):
    """Return a list of all files in the directory."""
    all_files = []
    for path, subdirs, files in os.walk(root):
        for name in files:
            all_files.append(os.path.join(path, name))
    return all_files

def insert_into_mongodb_client(client, data):
    if isinstance(data, list):
        client.insert_many(data, 'WIPO_BEFORE_2018')
    else:
        client.insert_many([data], 'WIPO_BEFORE_2018')
```

#### 2.3.1. For WIPO 2017 and before
Datasets for WIPO 2017 and before are shared in `OneDrive`. Currently the `.zip` files need to be manually downloaded and put into a folder `wipo_datasets` in the same directory with the executing python script.

[wipo_main.py](./wipo_main.py)

```
from web_crawler.extractor import Extractor
from dataset_parser.wipo.wipo_parser import WIPOParser
from db_clients.mongodb_client import MongoDBClient


if __name__ == "__main__":
    # Get ALL the files in folder `wipo_datasets`
    zip_dir = "wipo_datasets"
    zip_files = [filename for filename in iter(get_all_files_in_directory(zip_dir))]
    extractor = Extractor()
    datasets = []
    for filename in zip_files:
        datasets.extend(extractor.extract_zip(filename, zip_dir))

    parser = WIPOParser()
    client = MongoDBClient()
    parsed_data = []

    # Parse all the files in `wipo_datasets`
    for dataset in iter(datasets):
        parsed_data.append(parser.from_file(dataset))

        # Release the patents into MongoDB and clear the list
        if len(parsed_data) == 1000:
            insert_into_mongodb_client(client,parsed_data)
            parsed_data.clear()

    # Release the remaining patents
    if len(parsed_data) != 0:
        insert_into_mongodb_client(client, parsed_data)
```

#### 2.3.2. For WIPO 2018 in FTP server

Datasets for WIPO 2018 is on a FTP server. The `WIPOCrawler` is used to download all the datasets.

[wipo_main.py](./wipo_main.py)

```
from web_crawler.extractor import Extractor
from dataset_parser.wipo.wipo_parser import WIPOParser
from db_clients.mongodb_client import MongoDBClient


if __name__ == "__main__":
    # Only 2018 WIPO datasets require using WIPOCrawler
    crawler = WIPOCrawler(stay_alive=True)
    crawler.download_datasets()
    datasets = crawler.extract_compressed()
    crawler.close()

    parser = WIPOParser()
    client = MongoDBClient()
    parsed_data = []

    # Parse all the files in `wipo_datasets`
    for dataset in iter(datasets):
        parsed_data.append(parser.from_file(dataset))

        # Release the patents into MongoDB and clear the list
        if len(parsed_data) == 1000:
            insert_into_mongodb_client(client,parsed_data)
            parsed_data.clear()

    # Release the remaining patents
    if len(parsed_data) != 0:
        insert_into_mongodb_client(client, parsed_data)
```

___

## 3. Database Clients

### 3.1. MongoDB Client

#### Usage

[mongodb_client.py](./db_clients/mongodb_client.py)

```
from db_clients.mongodb_client import MongoDBClient


if __name__ == "__main__":
    sample_dict = {'patent-id': '6185736', 'application-date': '19970930', 'grant-date': '20010206', 'title': 'Information transmission apparatus, traffic control apparatus, method of', 'abstract': {'PAL': 'A notification parameter file, which time-sequentially shows acharacteristic of the transmission rate change corresponding to thedurable time of a traffic, is notified to a network from a servercomprising a storage medium storing data having a traffic characteristicensured at a transmission starting time. The network adds thecharacteristic of the transmission rate change designated by the notifiedparameter to the traffic characteristic already admitted so as todetermine whether or not the traffic is admitted. If the traffic can beadmitted, the network allocates a transmission bandwidth resource to thesever based on the characteristic of the transmission rate changedesignated by the notified parameter.'}, 'inventor': {'NAM': 'Ueno; Hideyuki', 'CTY': 'Tokyo'}, 'assignee': {'NAM': 'Kabushiki Kaisha Toshiba', 'CTY': 'Kawasaki', 'COD': '03'}, 'patent-citations': [{'PNO': '5854887', 'CNT': 'US'}, {'PNO': '5854887', 'CNT': 'US'}, {'PNO': '5854887', 'CNT': 'US'}, {'PNO': '5854887', 'CNT': 'US'}, {'PNO': '8-84339', 'CNT': 'JP'}], 'ipc-classifications': ['G06F 1516', 'H04N  7173', 'G08C 1500', 'H04J  316']}

    ## By default, connect to `localhost` on port `27017`
    client = MongoDBClient()


    ## Remove ALL documents in the collection
    #client.remove()


    ## Insert the sample object
    client.insert_many([sample_dict], 'USPTO1')


    ## Find the patents with correct attributes
    attributes = {
        'grant-date': '20010206'
    }
    patents = MongoDBClient().find(**attributes)
    print(patents[0]["title"])


    ## Print the number of patents in the collection
    print(client.count())


    ## Close the connection after using
    client.close()
```

___

## 4. Google

### 4.1. Google Patent Crawler

Crawl a single patent page on Google.
Example URL: https://patents.google.com/patent/CN1205428C/en

Without the suffix `/en`, the patent page will be in the ORIGINAL language of the patent.

The schema of the patent page can be found in variable `GOOGLE_PATENT_TAG_SCHEMA` in [google_patent_configs.py](./web_crawler/google/google_patent_configs.py).

The full list of custom operators (with explanations) on the attributes of the patent, such as `remove_all_whitespaces`, `remove_duplicates` and `only_get_first_tag`, can be found in variable `DEFAULT_SETTING_FOR_TAG` in [beautiful_soup_configs.py](./web_crawler/beautiful_soup_configs.py).

#### 4.1.1 Using with Tor

By default, the parameter `use_tor` is True, by which the crawler requires a running `Tor` instance to avoid exceeding rate limit to Google.

[stem](https://stem.torproject.org/index.html) is used as a controller to re-new the IP address after each crawling request.

First, for any script to talk with your relay, it will need to have a control port available. This is a port that's usually only available on `localhost` and protected by either a password or authentication cookie.

Look at your [torrc](https://www.torproject.org/docs/faq.html.en#torrc) for the following configuration options
```
## The port on which Tor will listen for local connections from Tor
## controller applications, as documented in control-spec.txt.
ControlPort 9051

## If you enable the controlport, be sure to enable one of these
## authentication methods, to prevent attackers from accessing it.
## This is the hashed for "password"
HashedControlPassword 16:AFCE8C15C40596ED6088A4D36A0BB70657424AE748141E5A3A813AB39A

#CookieAuthentication 1
```

#### 4.1.2 Crawl with publication numbers

Example number: `CN-1205428-C`

All dashes in the publication number will be removed upon crawling.

Parameter `translated_to_en` (Default: `True`) is only eligible for crawling with publication number. If `True`, the generated URLs will be added a suffix `/en`.

##### Usage

[google_patent_crawler.py](./web_crawler/google/google_patent_crawler.py)

```
from web_crawler.google.google_patent_crawler import GooglePatentCrawler


if __name__ == "__main__":
    ## 1. Crawl using only 1 application and publication_number pair
    app_pub_number = ('CN-00101078-A','CN-1205428-C')
    crawler = GooglePatentCrawler()
    output = crawler.crawl_from_application_and_publication_number(app_pub_number)


    ## 2. Crawl using a list of 1 application and publication_number pairs
    list_of_app_pub_number = [
        ('CN-00101078-A','CN-1205428-C'),
        ('CN-00105093-A','CN-1128624-C')
    ]
    crawler = GooglePatentCrawler(translated_to_en=False)
    output = crawler.crawl_from_application_and_publication_numbers(list_of_app_pub_number)
```

#### 4.1.3 Crawl with URLs

[google_patent_crawler.py](./web_crawler/google/google_patent_crawler.py)

```
from web_crawler.google.google_patent_crawler import GooglePatentCrawler


if __name__ == "__main__":
    ## 1. Crawl using only 1 URL
    url = "https://patents.google.com/patent/CN1205428C/en"
    crawler = GooglePatentCrawler(use_tor=False)
    output = crawler.crawl_from_url(url)


    ## 2. Crawl using a list of URLs
    list_of_urls = [
        "https://patents.google.com/patent/CN1205428C",
        "https://patents.google.com/patent/CN1128624C/en"
    ]
    crawler = GooglePatentCrawler()
    output = crawler.crawl_from_urls(list_of_urls)
```


___





## 5. Helper objects

### 5.1. FTP Client

Download files from a FTP server.  

The default download directory is `downloaded_ftp`.

Details of the downloaded files (including name, size and modified date) will be saved (in JSON) and compared upon the next download, to avoid re-downloading.

The client has the following optional keyword arguments:  

- download_dir: `str` - The directory to store the downloaded files.

- file_checking_function: `function` - The pointer to a function to check the necessary files to download. If not given, download all files.

- history_filename: `str` - The file to store all details of downloaded files.

##### Usage  
The `hostname` can be provided upon creating the `FTPClient` instance, or calling function `.connect`.

[ftp_client.py](./web_crawler/ftp_client.py)
```
from web_crawler.ftp_client import FTPClient


if __name__ == "__main__":
    hostname = '192.168.30.34'
    username = 'test_username'
    password = 'test_password'

    # Connect to the server
    client = FTPClient()
    client.connect(hostname)
    client.login(username, password)


    # After connecting, the current directory is `root`
    # get_files() and get_directories() return lists
    #  of files/directories in the CURRENT directory
    files = client.get_files()
    directories = client.get_directories()


    # To get files/directories in another path
    # Either change  the current directory to the path
    path = 'workspace/'
    files = client.get_files(path)
    directories = client.get_directories(path)

    #  or pass the path to get_files and get_directories
    client.cwd('workspace')
    files = client.get_files()
    directories = client.get_directories()


    # Close the connection
    client.close()
```

#### 4.1.1. file_checking_function

Internally, the details of the files in the FTP server are stored as a `FTPFile` object with only 3 attributes `name`: str, `size`:  int and `modified_date`: str.

The `file_checking_function` must have only 1 input - the `FTPFile` object to verify, and return True if it satisfies the conditions.

*Example*:
```
def is_correct_file(self, file_info: FTPFile):
    if file_info.name.endswith('_xml.tar.gz'):
        return True

    if file_info.size > 100: # Bytes
        return True

    return False

if __name__ == "__main__":
    hostname = "192.168.1.30"
    client = FTPClient(hostname, file_checking_function=is_correct_file)
```

#### 5.1.2. history_filename

If not given, by default, `history_filename` is None, which means the client won't store the details of downloaded files and will re-download all the files. This is good for one-time usage.  

Else, all the files will be compared to the auto-updated list of downloaded files.

*Example*:  
```
filename = "downloaded.json"
hostname = "192.168.1.30"
client = FTPClient(history_filename=filename)
```

### 5.2. Extractor

Extract Extract compressed files. Currently it only supports `.zip`, `.tar` and `.tar.gz` extensions.

#### Usage

[extractor.py](./web_crawler/extractor.py)

```
from web_crawler.extractor import Extractor


if __name__ == "__main__":

    # keep the original compressed files after extracting
    extractor = Extractor(remove_compressed=False)

    # Extract a .zip file to folder `extract_zip`
    zip_dir = "extract_zip"
    filename_zip = "test1.zip"
    extracted_zip = extractor.extract_zip(filename_zip, zip_dir))

    # Extract a .tar file to folder `extract_zip`
    tar_dir = "extract_tar"
    filename_tar = "test2.tar"
    extracted_tar = extractor.extract_zip(filename_tar, tar_dir))
```
