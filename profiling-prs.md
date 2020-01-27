---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.2'
      jupytext_version: 1.3.0
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# Exploring pull requests backlog with pygithub and plotly 

In this notebook we use `pygithub` to retrieve data about open pull requests of the scikit-image repository. Then we visualize wich modules and files have the largest number of open pull requests.

```python
from github import Github

# First create a Github instance:
# put your own token here to access Github API, see https://github.com/settings/tokens

g = Github("your_own_token_here")
repo = g.get_repo("scikit-image/scikit-image")
open_prs = repo.get_pulls(state='open')
```

For each file in an open pull request, we create a dictionary with some properties of the pull request. 

```python
all_files = []

for pr in open_prs:
    title = pr.title
    number = pr.number
    date = pr.created_at
    comments = pr.comments
    all_files += [{'filename':pr_file.filename, 'title':title, 
                   'number':number, 'date':date, 'comments':comments} 
                  for pr_file in pr.get_files()]

```

We put these data in a pandas dataframe, and do some data cleaning and massaging. It particular, we create a `module` column.

```python
import pandas as pd
import os
df = pd.DataFrame(all_files)
# Remove __init__.py
df = df[~df['filename'].str.endswith('__init__.py')]

df['module'] = [os.path.dirname(filename) for filename in df['filename']]
# Organization of files: test files should not have a separate submodule
df['module'] = [os.path.dirname(path) if path.endswith('tests') 
                else path for path in df['module']]
# Examples have their own module
df['module'] = ['examples' if path.startswith('doc/examples') 
                else path for path in df['module']]
df.sort_values(by='filename', inplace=True)
df['unit'] = 1
df['year']=df['date'].dt.year + df['date'].dt.month / 12.
df.loc[df.module.isna(), 'module'] = 'Other'
df.number = df.number.astype(str) # because of a bug to be corrected in px.sunburst

```

```python
df.columns
```

## Visualizations of files modified by pull request

Below we use the [plotly library](https://plot.ly/python/) to make several visualizations in order to understand which files are the most often modified by open pull requests. For this, we use hierarchical charts, namely [sunburst](https://plot.ly/python/sunburst-charts/) (Ubuntu users, think of Baobab) and [treemap](https://plot.ly/python/treemaps/) charts. These charts are interactive, you can click on a specific sector to zoom into it (and click on it again to go back). Note that the API used here was introduced in plotly 4.5 which was released very recently.

Colors of these charts can correspond to the upper level (here, `module` column) or to another column or array (which can be discrete or continuous). 

```python
import plotly.express as px
fig = px.sunburst(df, path=['module', 'filename', 'number'], values='unit',
                 hover_data=['title', 'date'], maxdepth=2)
fig.show()
```

In the treemap below we can explore the different modules, and see that for example `skimage/transform/_warps.py` and `skimage/measure/_regionprops.py` are modified in 10 open pull requests. Probably interesting to take a look at these pull requests together...

```python
import plotly.express as px
fig = px.treemap(df, path=['module', 'filename', 'number'], values='unit',
                 hover_data=['title', 'date'])
fig.show()
```

In the sunburst and treemap below (same data, you can choose the viz you prefer), we color the sectors by year where the PR was opened. For file and modules the color is the average of their leaves. With this kind of visualization you can see rapidly which PRs are very old or very recent, and in which modules there are located.

```python
fig = px.sunburst(df, path=['module', 'filename', 'number'], values='unit',
                 color='year', hover_data=['title', 'date'], maxdepth=2,
                 color_continuous_scale='rdbu')
fig.show()
```

```python
fig = px.treemap(df, path=['module', 'filename', 'number'], values='unit',
                 color='year', hover_data=['title', 'date'],
                 color_continuous_scale='rdbu')
fig.show()
```

Which pull requests received a lot of comments? Which were hardly reviewed?

```python
fig = px.treemap(df, path=['module', 'filename', 'number'], values='unit',
                 color='comments', hover_data=['title', 'date'],
                color_continuous_scale='rdbu_r')
fig.show()
```

## A more PR-centric approach

Above we were looking more at modules and files, looking for files which were often or less often targeted at by pull requests. Below we adopt a more pull-request-centric approach, in order to visualize pull requests more directly. For this we have to attribute a module to a pull request, which is not an easy definition since pull requests often modify files from several modules. As an approximation we choose the module with the largest number of files. 

```python
import os
from collections import Counter

def get_module(file_list):
    modules = []
    for file_iter in file_list:
        file_name = file_iter.filename
        module = os.path.dirname(file_name)
        if module.endswith('tests'):
            module = os.path.dirname(module)
        if module.startswith('doc/examples'):
            module = 'examples'
        modules.append(module)
    occurence_count = Counter(modules)
    return occurence_count.most_common(n=1)[0][0]
```

```python
all_files = []

for pr in open_prs:
    module = get_module(pr.get_files())
    all_files += [{'module':module, 'title':pr.title, 
                   'number':pr.number, 'date':pr.created_at, 
                   'comments':pr.comments, 'url':pr.url}] 
```

```python
df = pd.DataFrame(all_files)
df['year']=df['date'].dt.year + df['date'].dt.month / 12.
df.loc[df.module.isna(), 'module'] = 'Other'
df.number = df.number.astype(str) # because of a bug to be corrected in px.sunburst
```

```python
import plotly.express as px
fig = px.treemap(df, path=['module', 'number'],
                 hover_data=['title', 'date'])
fig.show()
```

```python
import plotly.express as px
fig = px.treemap(df, path=['module', 'number'],
                 hover_data=['title', 'date'], color='year',
                 color_continuous_scale='rdbu')
fig.show()
```

```python
import plotly.express as px
fig = px.treemap(df, path=['module', 'number'],
                 hover_data=['title', 'date'], color='comments',
                 color_continuous_scale='rdbu_r')
fig.show()
```

## Adding reviewers

New pull request modifying a specific file? Who can you ping to get the PR reviewed? The visualization below can help to get a feeling of who is already familiar with this piece of the code. For this we consider all pull requests, both closed and open, but we limit the number. 

Note that such large requests can overcome the authorized bandwidth of Github API!

```python
all_prs = repo.get_pulls(state='all')
all_files = []

i = 0
for pr in all_prs:
    if i > 1500:
        break
    title = pr.title
    number = pr.number
    date = pr.created_at
    comments = pr.comments
    url = pr.url
    contributors = set([comment.user.login for comment in pr.get_issue_comments()] + 
                       [pr.user.login])
    contributors.discard('pep8speaks')
    contributors.discard('codecov-io')
    files = pr.get_files()
    # Exclude some files so that the size of the viz is not enormous
    files = [fl.filename for fl in files 
                 if not (fl.filename.endswith('__init__.py') or
                                 fl.filename.endswith('setup.py') or
                                 'tests' in fl.filename)]
    all_files += [{'filename':pr_file, 'title':title, 
                   'number':number, 'date':date, 
                   'comments':comments, 'url':url, 'contributor':login} 
                  for pr_file in files for login in contributors]
    i += 1
```

```python
import pandas as pd
import os
df = pd.DataFrame(all_files)

df['module'] = [os.path.dirname(filename) for filename in df['filename']]
# Organization of files: test files should not have a separate submodule
df['module'] = [os.path.dirname(path) if path.endswith('tests') 
                else path for path in df['module']]
# Examples have their own module
df['module'] = ['examples' if path.startswith('doc/examples') 
                else path for path in df['module']]
#df.sort_values(by='filename', inplace=True)
df['unit'] = 1
df['year']=df['date'].dt.year + df['date'].dt.month / 12.
df.loc[df.module.isna(), 'module'] = 'Other'
df['filename'] = [os.path.basename(filename) for filename in df['filename']]
```

```python
import plotly.express as px
fig = px.treemap(df, path=['module', 'filename', 'contributor'])
fig.show()
```

```python
import plotly.express as px
fig = px.treemap(df, path=['module', 'filename', 'contributor'], color='year',
                color_continuous_scale='Inferno')
fig.show()
```

```python
df['year']
```

```python
df[:80]

```

```python

```
