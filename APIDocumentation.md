# API Documentation

## 1. USPTO

### 1.1. *web_crawler.uspto.uspto_lxml_crawler.*__USPTOCrawler__()

Download and extract the USPTO datasets from given URLs.

_Parameters_:  
- remove_compressed: `boolean` - Remove all downloaded compressed files after extracting them. Default: True.


__download_datasets__(*URLs*: List)  
    Return a list of extracted filenames after downloading files from the input list of URLs.


### 1.2. *dataset_parser.uspto.uspto_parser.*__USPTOParser__()
Parse a single text file and return a list of `Dict` (JSON) objects containing only the required information. This parser works for all 3 versions of USPTO datasets.

__parse__(*filename*: str)  
Return the parsed `Dict` object from the input file.

__write_to_json_file__(*jData*: Dict, *filename*=None, *output_directory*='parsed_datasets', *pretty*=False, *skipkeys*=True, *ensure_ascii*=False)  
Serialize obj to a JSON formatted str using this conversion table.

- If no `filename` is specified, the `__class__.__name__` of the parser object is used instead.  

- If `skipkeys` is True (default: True), then dict keys that are not of a basic type (str, int, float, bool, None) will be skipped instead of raising a TypeError.

- If `ensure_ascii` is True (default: False), the output is guaranteed to have all incoming non-ASCII characters escaped. If ensure_ascii is false, these characters will be output as-is.

- If `pretty` is True (default: False), the JSON array elements and object members will be pretty-printed with indent level 2.

___

## 2. WIPO

### 2.1. *web_crawler.wipo.wipo_crawler.*__WIPOCrawler__(*hostname*: str=None, *username*: str=None, *password*: str=None)

Download the datasets for WIPO 2018 on a FTP server.

_Parameters_:  

- hostname, username, password: `str` - the credentials to connect to the FTP server. If all are `None`, [.env](./web_crawler/wipo/.env) will be used as default.

- remove_compressed: `boolean` - Remove all downloaded compressed files after extracting them. Default: True.

- stay_alive: `boolean`. Default: False
    - True:   A connection to the server is created and maintained until `self.close()` is MANUALLY called. Note that the WIPO FTP server automatically closes the connection after long idling. Unless the connection is used continuously, `stay_alive=False` is the preferred.
    - False:  Only starts a connection upon a download function is called, causing an overhead for each request. This is to politely connect/close the connections with the server for all requests.

- history_filename: `str` - The file containing the information of the downloaded datasets. Default: `wipo_downloaded.json`.

- output_directory: `str` - The directory for downloading, extracting and processing datasets. Default: `wipo_datasets`.

__download_datasets__(*path*: str = '')  
    Recursively download only the WIPO datasets in the given `path` and its sub-directories on the FTP server.

__extract_compressed__(*directory*: str = None)  
    Extract the downloaded datasets to the given `directory`. If no `directory` is given, the files will be extracted to the default `self.output_directory` (which is `wipo_datasets`).

__close__()  
    Close the connection with the WIPO server.


### 2.2. *dataset_parser.wipo.wipo_parser.*__WIPOParser__()
Parse a single text file and return a `Dict` (JSON) object containing only the required information. This parser works for both versions of WIPO datasets.

__from_string__(*text*: str)  
    Parse a XML string into a `Dict` object.

__from_file__(*filename*: str)  
    Parse a XML file into a `Dict` object.

__is_2018_dataset__(*text*: str)  
    Return `True` if the given `text` is a WIPO 2018 dataset.


__write_to_json_file__(*jData*: Dict, *filename*=None, *output_directory*='parsed_datasets', *pretty*=False, *skipkeys*=True, *ensure_ascii*=False)  
    Serialize obj to a JSON formatted str using this conversion table.

- If no `filename` is specified, the `__class__.__name__` of the parser object is used instead.  

- If `skipkeys` is True (default: True), then dict keys that are not of a basic type (str, int, float, bool, None) will be skipped instead of raising a TypeError.

- If `ensure_ascii` is True (default: False), the output is guaranteed to have all incoming non-ASCII characters escaped. If ensure_ascii is false, these characters will be output as-is.

- If `pretty` is True (default: False), the JSON array elements and object members will be pretty-printed with indent level 2.

___

## 3. Database Clients

### 3.1. *db_clients.mongodb_client.*__MongoDBClient__(*database*, *collection*=Patent)

_Parameters_:  

- database: `str` - name of the database to connect to in mongo.

- collection: `mongoengine document` - a class that inherits from `mongoengine.Document`. Read more about creating one [here](http://docs.mongoengine.org/guide/defining-documents.html). Default: [Patent](./db_clients.patent_format.py).

All other keyword arguments will be passed directly to [mongoengine.connect()](http://docs.mongoengine.org/guide/connecting.html).

By default, MongoEngine assumes that the mongod instance is running on `localhost` on port `27017`. If MongoDB is running elsewhere, you should provide the `host` and `port` arguments:
```
client = MongoDBClient('innosenze_development', host='192.168.1.35', port=12345)
```

__close__()  
Close the connection with the MongoDB. A new instance of this client needs to be created for further usage.


__insert_many__(*objects*: List, *source*: str)  
Replace all dots and dollar signs in all keys in <Dict> objects and insert them into MongoDB.

- `source` is an additional attribute for all patents, to indicate where the patent is taken from. For example: `USPTO1`, `USPTO2`, `WIPO_2018` , etc.

- MongoDB doesn't allow `$` or `.` characters as map keys due to [restrictions on field names](https://docs.mongodb.com/manual/reference/limits/#Restrictions-on-Field-Names).

__find__(_**kargs_)  
Return a list of objects from passing the keyword arguments to `object` attribute of the `Document` collection.

More queries can be read from [here](http://docs.mongoengine.org/guide/querying.html).

```
attributes = {
    'grant-date': '20010206'
}
patents = MongoDBClient().find(**attributes)
```

__remove__(_**kargs_)  
Remove the documents satisfying the conditions in `kargs` in the collection. USE AT OWN RISK!  
- If no `kargs` is provided, all documents will be removed.

__count__()  
Return the number of documents in the collection.

___

## 4. Google

___


## 5. Helper objects

### 5.1. *web_crawler.ftp_client.*__FTPClient__(*hostname*: str=None, *download_dir*: str="downloaded_ftp", *file_checking_function*: function=None, *history_filename*: str=None)

Download files from a FTP server.  

Internally, the details of the files in the FTP server are stored as a `FTPFile` object with only 3 attributes `name`: str, `size`:  int and `modified_date`: str.

*Parameters*:  

- hostname: `str` - The hostname to connect to. If not given, `connect(hostname)` can be used after the `FTPClient` has been created. Default : `None`.

- download_dir: `str` - The directory to store the downloaded files. Default: `downloaded_ftp`.

- file_checking_function: `function` - The pointer to a function to check the necessary files to download. If not given, download all files. Default: `None`.
    - The `file_checking_function` must have only 1 input - the `FTPFile` object to verify, and return `True` if it satisfies the conditions.  


- history_filename: `str` - The file to store all details of downloaded files. Default: `None`.
    - If not given, by default, `history_filename` is None, which means the client won't store the details of downloaded files and will re-download all the files. This is good for one-time usage.  


__connect__(*hostname*: str)  
Connect to the FTP server in `hostname`.

__login__(*username*: str=''. *password*: str='')  
Login to the server using the given `username` and `password`, if necessary. This function must be called after `connect(hostname)`.

__close__()  
Politely close the connection with the FTP server. New connections can be made with `connect(hostname)` after closing the previous one.

__get_files__(*path*: str='')  
Return a list of files in the current working directory if `path` is empty; otherwise, return a list of files in the given `path`.

__get_directories__(*path*: str='')  
Return a list of directories in the current working directory if `path` is empty; otherwise, return a list of directories in the given `path`.

__cwd__(*path*: str)  
Change the current working directory to `path`.

__download_files__(*path*: str='')
Download the files in the given `path`. If `path` is empty, download the files in the current working directory.

### 5.2. *web_crawler.extractor.*__Extractor__(*remove_compressed*: bool=False)

Extract compressed files. Currently it only supports `.zip`, `.tar` and `.tar.gz` extensions.

*Parameters*:  

- remove_compressed: `bool` - If True, remove the orignally compressed files after extracting them. Default : `False`.

__extract_zip__(*filename*: str, *output_directory*: str='')  
Extract the files in a `.zip` file to `output_directory`. If no `output_directory` is given, extract the files to the current  directory.

__extract_tar__(*filename*: str, *output_directory*: str='')  
Extract the files in a `.tar` or `tar.gz` file to `output_directory`. If no `output_directory` is given, extract the files to the current  directory.

__remove_file__(*filename*: str)  
Use `os.remove` to delete the given file.
