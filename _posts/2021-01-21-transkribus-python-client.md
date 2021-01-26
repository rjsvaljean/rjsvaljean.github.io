---
title: "Using The Python Transkribus Client"
published: true
---

Introduction
------------

This will be the first in a series of posts on various little bits of technology I had to learn in the course of completing my Master's thesis: "Automated Transcription of Alexander Carmichael's Field Notebooks". The target audience of this post are researchers working on transcription projects, using Transkribus who aren't software developers but do have some knowledge of git and python.

One of the steps in the project was assembling a dataset for training a handwritten text recognition (HTR) model from the notebooks and TEI-XML (Text Encoding Initiative XML) encoded transcriptions of Alexander Carmichael's (a 19th century scottish folklorist) field notebooks. And it turns out that one of the many many things Transkribus does well is being a good annotation store. So, if you're interested in interacting with the documents and annotations you have stored on Transkribus programatically, read on.

Background
----------

The steps for doing this were:

1. Detecting text regions in the image and drawing bounding polygons around them.
2. Annotating these bounding polygons with text drawn from the TEI encoded transcriptions in Transkribus (for discoverability).
3. Extracting these annotations from Transkribus.

Step 1 is done pretty well by Transkribus. Step 2 involved some XML transformation and manual entry. Step 3 is where Transkribus' HTTP API and the python client to it come in. Transkribus is a great tool for working with handwritten documents in general apart from its HTR capabilities. It has a solid document and page model and using the PAGE XML format for storing annotations. So, it works great just as a store for your scanned pages and their transcriptions. While its primary interface is the GUI, it was very wisely built on top of a HTTP API which can be interfaced with programatically.

Get to the code already
-----------------------

Alright, alright. Like a bunch of kids in a Stephen King novel I haven't read, let's get to It. I used the [TranskribusPyClient](https://github.com/Transkribus/TranskribusPyClient/) library. It seems like it is intended to be used as a command line tool but it is also quite possible to use it as a library in your application. It seems pretty well maintained. Check the repository out into your workspace and then you should be able to import the client class using:

```
from TranskribusPyClient.src.TranskribusPyClient.client import TranskribusClient
```

There is a better way to install it using pip but the tool doesn't seem to offer one and that isn't the focus of this blog.

Now that you have the class, you can instantiate it using:

```
client = TranskribusClient()
```

And login using

```
client.auth_login('my_username', 'my_password')
```

You can see all the collections you have using:

```
client.listCollections()
# [(67146, 'Alexander Carmichael Field Notebooks', 4)]
```

To see the documents in the above collection:

```
docs = client.listDocsByCollectionId(67146)
```

Let's get some details out of it:

```
doc_ids_and_page_counts_by_name = dict(map(lambda x: (x['title'], (x['docId'], x['nrOfPages'])), docs))
doc_ids_and_page_counts_by_name

# {'pages': (400767, 589),
#  'TRAINING_VALIDATION_SET_Alexander Carmichael HTR': (411716, 28),
#  'Sample_TRAINING_VALIDATION_SET_Alexander Carmichael HTR': (449417, 17),
#  'TRAINING_VALIDATION_SET_Carmichael Model w/out Base Model': (449558, 24)}
```

`400767` is the document containing all the pages from the notebooks and `411716` is a subset created for validation.

Using these identifiers we can start getting data from the page level. For instance, here is some code I wrote to calculate character error rates (CER) for the validation set:

```
validation_set = client.getDocById(67146, 411716) # the first argument is the collection id and the second is the document id

def get_cer(texts):
    text1, text2 = texts
    return float(distance(text1, text2)) / len(text2)

ranked_transcriptions = []
for page in validation_set['pageList']['pages']: # Iterate through all pages in the documents
    transcriptions = page['tsList']['transcripts'] # Get all the transcriptions attached to a page
    automated_transcription = list(filter(lambda t: t['status'] == 'IN_PROGRESS', transcriptions))[0] # The IN_PROGRESS transcriptions were the ones that Transkribus' HTR model generated
    ground_truth_transcription = list(filter(lambda t: t['status'] == 'GT', transcriptions))[0] # And GT transcriptions were the the ground truth
    a_soup = BeautifulSoup(requests.get(automated_transcription['url']).text) # Use Beautiful Soup to parse the PAGE XML and the lines of text should be within <textline> tags
    gt_soup = BeautifulSoup(requests.get(ground_truth_transcription['url']).text)
    transcriptions = list(map(lambda x: (x[0].find('unicode').text, x[1].find('unicode').text),
             zip(a_soup.find_all('textline'), gt_soup.find_all('textline'))))
    cers_of_transcriptions = zip(map(get_cer, transcriptions), transcriptions) # Compute CER for each pair of automated and ground truth transcription
    for cer, (automated, ground_truth) in cers_of_transcriptions:
        ranked_transcriptions.append((cer, automated, ground_truth))
```

Conclusion
----------

This is just a little step into a simple use case I had for the Transkribus REST API. Nothing complicated at all but I wanted to put it out here in case it might save otheres having to do some repetitive pointing and clicking that could be automated away using this REST API. TranskribusPyClient covers a lot more functionality that you can see documented [here](https://github.com/Transkribus/TranskribusPyClient/wiki) and an introductory paper [here](https://europe.naverlabs.com/wp-content/uploads/ultimatemember/temp/2383a0b295c90339a809ea1b5ebeda8789458d38.pdf). More details about the workflow involved in my thesis project will be included in following posts.
