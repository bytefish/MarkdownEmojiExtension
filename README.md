# MarkdownEmojiExtension #

I didn't have much time this weekend, but I wanted to write a little code for keeping my brain 
active. So I ego-googled my website and noticed: I can't use emojis! 

Supporting emojis sounds like a nice little task, so let's take a look how to do it.

You can find the final extensions on the GitHub repository at:

* [https://github.com/bytefish/MarkdownEmojiExtension](https://github.com/bytefish/MarkdownEmojiExtension)

## Pelican ##

My website is generated by Pelican, which is a static site generator written in Python:

* [https://github.com/getpelican](https://github.com/getpelican)

I am using Markdown for writing the posts, so the plan is to write a custom Markdown extension 
like described in the Python Markdown docs and hook an extension into the HTML generation:

* [Tutorial: Writing Extensions for Python Markdown](https://github.com/Python-Markdown/markdown/wiki/Tutorial:-Writing-Extensions-for-Python-Markdown)

## Emojis ##

Next I got a list of Emojis from the [EmojiCodeSheet] project, which is located at:

* [https://github.com/shanraisshan/EmojiCodeSheet](https://github.com/shanraisshan/EmojiCodeSheet)

Among many other files, there are several JSON files, with the emojis as Unicode characters:

```
{
  "peoples": {
    "people": [
      {
        "key": "grinning_face",
        "value": "😀"
      },
      {
        "key": "grimacing_face",
        "value": "😬"
      },
      
      ...
```

The plan is to directly use the key for Markdown and include emojis like this: ``::grinning_face::``. I then edited the 
JSON file, so it boils down to a flat list of Key (Emoji Name) / Value (Actual Emoji) pairs. The ``emojis.json`` file with 
the definition is saved to a file ``emojis.json``.

## Implementing the Markdown Extension ##

First of all I implemented the Markdown extension. I commented it thoroughly so there is no need to describe it 
in detail. The basic idea is to read the ``emoji.json`` list of key / value pairs and turn them into a ``dict`` 
for faster lookups, these will be passed into a ``Pattern`` implementation, that handles matches for the Emoji 
Regexp (``::emoji_name::``).

```python
from markdown.extensions import Extension
from markdown.inlinepatterns import Pattern

from pkg_resources import resource_stream, resource_listdir

import json
import io
import os
import codecs

EMOJI_RE = r'(::)(.*?)::'

class EmojiExtension(Extension):

    def __init__(self, **kwargs):
        super(EmojiExtension, self).__init__(**kwargs)
                
    def extendMarkdown(self, md, md_globals):
        # Read the Emojis from the Resource:
        emoji_list = json.loads(resource_stream('resources', 'emojis.json').read().decode('utf-8'))
        # Turn the Emojis into a Dictionary for faster lookups:
        emojis = dict((emoji['key'], emoji['value']) for emoji in emoji_list)
        # And add the EmojiInlineProcessor to the Markdown Pipeline:
        md.inlinePatterns.add('emoji', EmojiInlineProcessor(EMOJI_RE, emojis) ,'<not_strong')
        
class EmojiInlineProcessor(Pattern):
    
    def __init__(self, pattern, emojis):
        super(EmojiInlineProcessor, self).__init__(pattern)
        
        self.emojis = emojis
        
    def handleMatch(self, m):
        emoji_key = m.group(3)
        
        return self.emojis.get(emoji_key, '')
```

Finally I wrote a ``setup.py`` file to install the Extension:

```python
from setuptools import setup, find_packages

setup(
    name='emojiextension',
    description='Extension for displaying Emojis',
    version='1.0',
    py_modules=['emojiextension'],
    install_requires = ['markdown>=2.5'],
    packages=find_packages(),
    package_data={'': ['*.json']},
    include_package_data=True
)
```

The extension can now be installed by running:

<pre>
python setup.py install
</pre>

## Integrating it into Pelican ##

What's left is to integrate the Markdown extension into Pelican. So in the Pelican Settings file I am first importing the ``EmojiExtension``:

```python
from emojiextension import EmojiExtension
```

And then register it in the ``MARKDOWN`` variable:

```python
# Markdown Configuration:
MARKDOWN = {
    'extensions' : [EmojiExtension()],
    'extension_configs': {
        'markdown.extensions.codehilite': {'css_class': 'highlight'},
        'markdown.extensions.toc' : {},
        'markdown.extensions.extra': {},
        'markdown.extensions.meta': {},
    },
    'output_format': 'html5',
}
```

And that's it!