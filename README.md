# Text_Classification
Text classification for UtaPass and KKBOX total reviews using different machine learning models.

## Introduction
This analysis is based on text data of UtaPass and KKBOX reviews on Google Play platform. As a KKStreamer from KKBOX, I have always wanted to analyze and classifify the polarity on app reviews. Concretely, I crawled the data using web crawler technique, which is an Internet bot that systematically browses the World Wide Web, and further using different deep learning models (Simple RNN, LSTM, Bi-directional LSTM, GRU, and CNN_LSTM).

## Data Source
1. [UtaPass reviews on Google Play](https://play.google.com/store/apps/details?id=com.kddi.android.UtaPass&hl=ja&showAllReviews=true)
2. [KKBOX reviews on Google Play](https://play.google.com/store/apps/details?id=com.skysoft.kkbox.android&hl=ja&showAllReviews=true)

## Research Questions and Bottleneck
* Do every reviews have sentiment words or charateristic of polarity?
* Do text pre-processing (remove stop words, remove punctuation, remove bad characters) be neccessary? 
* Is there any useless , redundant or even invalid information about the reviews? Do we need to utilize the method such as Anomaly Detection?
* Is there any online user make fake comments to affect other online users on the net? 
  > Luca shows that when a product or business has increased the +1 star rating, it increases revenue by 5-9%. Due to the financial benefits associated with online reviews, paid or prejudiced reviewers write fake reviews to mislead a product or business. [[M.Luca, “Reviews, reputation, and revenue: The case of yelp.com,” Harvard Business School
Working Papers, 2011](https://www.hbs.edu/faculty/Publication%20Files/12-016_a7e4a5a2-03f9-490d-b093-8f951238dba2.pdf)]

## Preparation
1. Preparing [selenium](https://pypi.org/project/selenium/), [beautiful soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/), and [pandas](https://pandas.pydata.org/pandas-docs/stable/install.html).
```import time
from bs4 import BeautifulSoup
import sys, io
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.proxy import *
import pandas as pd
```
2. Doing text pre-processing after installing [MeCab](https://pypi.org/project/mecab-python-windows/), [neologdn](https://pypi.org/project/neologdn/), [re](https://docs.python.org/3.6/library/re.html), and [emoji](https://pypi.org/project/emoji/)
```
import MeCab
from os import path
import neologdn
import re
import emoji

pos_list = [10, 11, 31, 32, 34]
pos_list.extend(list(range(36,50)))
pos_list.extend([59, 60, 62, 67])

def create_mecab_list(text):
    mecab_list = []
    mecab = MeCab.Tagger("-Ochasen")
    mecab.parse("")
    # encoding = text.encode('utf-8')
    node = mecab.parseToNode(text)
    while node:
        if len(node.surface) > 1:
            if node.posid in pos_list:
                morpheme = node.surface
                mecab_list.append(morpheme)
        node = node.next
    return mecab_list

def give_emoji_free_text(text):
    allchars = [str for str in text]
    emoji_list = [c for c in allchars if c in emoji.UNICODE_EMOJI]
    cleaned_text = ' '.join([str for str in text.split() if not any(i in str for i in emoji_list)])
    return cleaned_text

def clean_text(text):
    text = give_emoji_free_text(text)
    text = neologdn.normalize(text)
    text = create_mecab_list(text)    
    return text
```
3. Since the page at Google Play has to scroll down and click the "see more" button to view the whole reviews, I have to set a function to cope with these problems.
```
no_of_reviews = 1000
non_bmp_map = dict.fromkeys(range(0x10000, sys.maxunicode + 1), 0xfffd)

from selenium.common.exceptions import NoSuchElementException        
def check_exists_by_xpath(xpath):
    try:
        driver.find_element_by_xpath(xpath)
    except NoSuchElementException:
        return False
    return True
reviews = pd.DataFrame(columns = ["review", "Author Name", "Review Date", "Review Ratings", 
                                  "Review Body", "Developer Reply"])
temp = {"review": 0, "Author Name": "", "Review Date": "", "Review Ratings": 0, 
        "Review Body": "", "Developer Reply": ""}

def replace_value_with_definition(key_to_find, definition):
    for key in temp.keys():
        if key == key_to_find:
            temp[key] = definition
```
4. Start crawling the web (Reference: https://github.com/ranjeet867/google-play-crawler)
```
driver = webdriver.Chrome(r"./chromedriver")
wait = WebDriverWait(driver, 10)

# Append your app store urls here
urls = ["https://play.google.com/store/apps/details?id=com.kddi.android.UtaPass&hl=ja"]

for url in urls:

    driver.get(url)

    page = driver.page_source

    soup_expatistan = BeautifulSoup(page, "html.parser")

    expatistan_table = soup_expatistan.find("h1", class_="AHFaub")

    print("App name: ", expatistan_table.string)

    expatistan_table = soup_expatistan.findAll("span", class_="htlgb")[4]

    print("Installs Range: ", expatistan_table.string)

    expatistan_table = soup_expatistan.find("meta", itemprop="ratingValue")

    print("Rating Value: ", expatistan_table['content'])

    expatistan_table = soup_expatistan.find("meta", itemprop="reviewCount")

    print("Reviews Count: ", expatistan_table['content'])

    soup_histogram = soup_expatistan.find("div", class_="VEF2C")

    rating_bars = soup_histogram.find_all('div', class_="mMF0fd")

    for rating_bar in rating_bars:
        print("Rating: ", rating_bar.find("span").text)
        print("Rating count: ", rating_bar.find("span", class_="L2o20d").get('title'))

    # open all reviews
    url = url + '&showAllReviews=true'
    driver.get(url)
    time.sleep(5) # wait dom ready
    for i in range(1,25):
        try:
            driver.execute_script('window.scrollTo(0, document.body.scrollHeight);') # scroll to load other reviews
            time.sleep(2)
            if check_exists_by_xpath('//*[@id="fcxH9b"]/div[4]/c-wiz[2]/div/div[2]/div/div[1]/div/div/div[1]/div[2]/div[2]/div/content/span'):
                driver.find_element_by_xpath('//*[@id="fcxH9b"]/div[4]/c-wiz[2]/div/div[2]/div/div[1]/div/div/div[1]/div[2]/div[2]/div/content/span').click()
                time.sleep(2)
        except:
            pass

    page = driver.page_source

    soup_expatistan = BeautifulSoup(page, "html.parser")
    expand_pages = soup_expatistan.findAll("div", class_="d15Mdf")
    counter = 1
    items = []
    
    for expand_page in expand_pages:
        try:
            # print("\n===========\n")

            review = str(counter)
            
            Author_Name = str(expand_page.find("span", class_="X43Kjb").text)
            
            Review_Date = str(expand_page.find("span", class_="p2TkOb").text)
            
            reviewer_ratings = expand_page.find("div", class_="pf5lIe").find_next()['aria-label'];
            reviewer_ratings = reviewer_ratings.split('/')[0]
            reviewer_ratings = ''.join(x for x in reviewer_ratings if x.isdigit())
            Reviewer_Ratings = int(reviewer_ratings)

            Review_Body = str(expand_page.find("div", class_="UD7Dzf").text)
            Review_Body_cleaned = clean_text(Review_Body)
            Review_Body_string = ''.join(Review_Body_cleaned)
            
            developer_reply = expand_page.find_parent().find("div", class_="LVQB0b")
            if hasattr(developer_reply, "text"):
                Developer_Reply = str(developer_reply.text)
            else:
                Developer_Reply = ""
            
            counter += 1           
            item = {
                    "review": counter - 1,
                    "Author Name": Author_Name,
                    "Reviewer Ratings": Reviewer_Ratings,
                    "Review Date": Review_Date,
                    "Review Body": Review_Body_string,
                    "Developer Reply": Developer_Reply
                    }
            items.append(item)
                
        except:
            pass
driver.quit()
```
5. Transforming the data into dataframe using pandas, and removing the rows which contain empty cell.
```
df = pd.DataFrame(items, columns = ["review", "Author Name", "Review Date", "Reviewer Ratings", 
                                    "Review Body", "Developer Reply"])
                                    import numpy as np
df['Review Body'].replace('', np.nan, inplace=True)
df.dropna(subset=['Review Body'], inplace=True)
```
![GitHub Logo](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%883.38.26.png)
6. Finally, combine KKBOX reviews dataframe and UtaPass dataframe~ There would be 2250 reviews over two dataset.

## Let the rob hit the road!
1. We first start by loading the raw data. Each textual reviews is splitted into a positive part and a negative part. We group them together in order to start with only raw text data and no other information. If the reviewer rating is lower than 3 stars, we will divide it into the negative group. 
```
df = pd.read_csv("reviews_kkstream.csv")
import numpy as np

# create the label
df["is_bad_review"] = df["Reviewer Ratings"].apply(lambda x: 0 if int(x) <= 3 else 1)
# select only relevant columns
df = df[["Review Body", "is_bad_review"]]
df.head()
```
![GitHub Logo](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%884.19.52.png)

2. Split the data into training data and testing data
* Training set: a subset to train a model
* Testing set: a subset to test the trained model

```
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.model_selection import train_test_split

sentences = df['Review Body'].apply(str).values
y = df['is_bad_review'].values

sentences_train, sentences_test, y_train, y_test = train_test_split(sentences, y, test_size=0.20, random_state=1000)
```
![Overfitting/Underfitting a Model](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%885.04.17.png)

3. Import the packages we need
```
import tensorflow as tf
import numpy
from keras.models import Sequential,Model
from keras.layers import Dense, Dropout, Activation, Flatten
from keras.layers import Embedding
from keras.layers import LSTM,Bidirectional
from keras.layers import SimpleRNN
from keras.layers import GRU
from keras.layers import Convolution1D, MaxPooling1D
from keras.engine import Input
from keras.optimizers import SGD
from keras.preprocessing import text,sequence
import pandas
import os
from gensim.models.word2vec import Word2Vec
```

4. Set all the parameters
```
# Input parameters
max_features = 5000
max_len = 200
embedding_size = 300

# Convolution parameters
filter_length = 3
nb_filter = 150
pool_length = 2
cnn_activation = 'relu'
border_mode = 'same'

# RNN parameters
output_size = 50
rnn_activation = 'tanh'
recurrent_activation = 'hard_sigmoid'

# Compile parameters
loss = 'binary_crossentropy'
optimizer = 'rmsprop'

# Training parameters
batch_size = 128
nb_epoch = 250
validation_split = 0.25
shuffle = True
```

5. Build the word2vec model to do word embedding. (Reference: https://github.com/philipperemy/japanese-words-to-vectors/blob/master/README.md)
```
# Build vocabulary & sequences
tk = text.Tokenizer(nb_words=max_features, filters='!"#$%&()*+,-./:;<=>?@[\\]^_`{|}~\t\n', lower=True, split=" ")
tk.fit_on_texts(sentences)
x = tk.texts_to_sequences(sentences)
word_index = tk.word_index
x = sequence.pad_sequences(x,maxlen=max_len)

# Build pre-trained embedding layer
import gensim
w2v = Word2Vec.load('ja-gensim.50d.data.model')

from collections import Counter

word_vectors = w2v.wv
MAX_NB_WORDS = len(word_vectors.vocab)
MAX_SEQUENCE_LENGTH = 200
WV_DIM = 50
nb_words = min(MAX_NB_WORDS, len(word_vectors.vocab))
vocab = Counter()
word_index = {t[0]: i+1 for i,t in enumerate(vocab.most_common(MAX_NB_WORDS))}

# we initialize the matrix with random numbers
import numpy as np
wv_matrix = (np.random.rand(nb_words, WV_DIM) - 0.5) / 5.0
for word, i in word_index.items():
    if i >= MAX_NB_WORDS:
        continue
    try:
        embedding_vector = word_vectors[word]
        # words not found in embedding index will be all-zeros.
        wv_matrix[i] = embedding_vector
    except:
        pass      

import tensorflow as tf
from keras.layers import Dense, Input, CuDNNLSTM, Embedding, Dropout,SpatialDropout1D, Bidirectional
from keras.models import Model
from keras.optimizers import Adam
from keras.layers.normalization import BatchNormalization

embedding_layer = Embedding(nb_words, 
                     WV_DIM, 
                     mask_zero = False, 
                     weights = [wv_matrix], 
                     input_length = MAX_SEQUENCE_LENGTH, 
                     trainable = False)
```

6. Construct the five models.
Reference: [amazon-sentiment-keras-experiment](https://github.com/asanilta/amazon-sentiment-keras-experiment), [img2txt(CNN+LSTM)](https://github.com/teratsyk/bokete-ai)
* Simple RNN
```
# Simple RNN

model_RNN = Sequential()
model_RNN.add(embedding_layer)
model_RNN.add(SimpleRNN(output_dim=output_size, activation=rnn_activation))
model_RNN.add(Dropout(0.25))
model_RNN.add(Dense(1))
model_RNN.add(Activation('sigmoid'))

model_RNN.compile(loss=loss,
                  optimizer=optimizer,
                  metrics=['accuracy'])

print('Simple RNN')

from keras.callbacks import EarlyStopping, TensorBoard, ModelCheckpoint

path = 'weights.{epoch:02d}-{loss:.2f}-{acc:.2f}.hdf5'
model_checkpoint = ModelCheckpoint(path, monitor = 'loss', verbose = 1, save_best_only = True, mode = 'auto')
early_stopping = EarlyStopping(monitor='loss', patience = 8, verbose = 1, mode = 'auto')

history_RNN = model_RNN.fit(x, y, batch_size = batch_size, 
                            epochs = nb_epoch, 
                            validation_split = validation_split, 
                            shuffle = shuffle, 
                            verbose = 1, 
                            callbacks = [model_checkpoint, early_stopping])
```
* GRU
```
# GRU

model_GRU = Sequential()
model_GRU.add(embedding_layer)
model_GRU.add(GRU(units = output_size, activation = rnn_activation,recurrent_activation = recurrent_activation))
model_GRU.add(Dropout(0.25))
model_GRU.add(Dense(1))
model_GRU.add(Activation('sigmoid'))

model_GRU.compile(loss = loss,
                  optimizer = optimizer,
                  metrics = ['accuracy'])

print('GRU')

from keras.callbacks import EarlyStopping, TensorBoard, ModelCheckpoint

path = 'weights.{epoch:02d}-{loss:.2f}-{acc:.2f}.hdf5'
model_checkpoint = ModelCheckpoint(path, monitor = 'loss', verbose = 1, save_best_only = True, mode = 'auto')
early_stopping = EarlyStopping(monitor='loss', patience = 8, verbose = 1, mode = 'auto')

history_GRU = model_GRU.fit(x, y, batch_size = batch_size, 
                          epochs = nb_epoch, 
                          validation_split = validation_split, 
                          shuffle = shuffle, 
                          verbose = 1, 
                          callbacks = [model_checkpoint, early_stopping])
```
* LSTM
```
# LSTM

model_LSTM = Sequential()
model_LSTM.add(embedding_layer)
model_LSTM.add(Dropout(0.5))
model_LSTM.add(LSTM(units = output_size, activation = rnn_activation, recurrent_activation = recurrent_activation))
model_LSTM.add(Dropout(0.25))
model_LSTM.add(Dense(1))
model_LSTM.add(Activation('sigmoid'))

model_LSTM.compile(loss=loss,
                   optimizer=optimizer,
                   metrics=['accuracy'])

print('LSTM')

from keras.callbacks import EarlyStopping, TensorBoard, ModelCheckpoint

path = 'weights.{epoch:02d}-{loss:.2f}-{acc:.2f}.hdf5'
model_checkpoint = ModelCheckpoint(path, monitor = 'loss', verbose = 1, save_best_only = True, mode = 'auto')
early_stopping = EarlyStopping(monitor='loss', patience = 8, verbose = 1, mode = 'auto')

history_LSTM = model_LSTM.fit(x, y, batch_size = batch_size, 
                              epochs = nb_epoch, 
                              validation_split = validation_split, 
                              shuffle = shuffle, 
                              verbose = 1, 
                              callbacks = [model_checkpoint, early_stopping])
```
* BiLSTM
```
# Bidirectional LSTM

model_BiLSTM = Sequential()
model_BiLSTM.add(embedding_layer)
model_BiLSTM.add(Bidirectional(LSTM(units=output_size,activation=rnn_activation,recurrent_activation=recurrent_activation)))
model_BiLSTM.add(Dropout(0.25))
model_BiLSTM.add(Dense(1))
model_BiLSTM.add(Activation('sigmoid'))

model_BiLSTM.compile(loss=loss,
                     optimizer=optimizer,
                     metrics=['accuracy'])

print('Bidirectional LSTM')

from keras.callbacks import EarlyStopping, TensorBoard, ModelCheckpoint

path = 'weights.{epoch:02d}-{loss:.2f}-{acc:.2f}.hdf5'
model_checkpoint = ModelCheckpoint(path, monitor = 'loss', verbose = 1, save_best_only = True, mode = 'auto')
early_stopping = EarlyStopping(monitor='loss', patience = 8, verbose = 1, mode = 'auto')

history_BiLSTM = model_BiLSTM.fit(x, y, batch_size = batch_size, 
                                  epochs = nb_epoch, 
                                  validation_split = validation_split, 
                                  shuffle = shuffle, 
                                  verbose = 1, 
                                  callbacks = [model_checkpoint, early_stopping])
```
* CNN + LSTM (Based on "[Convolutional Neural Networks for Sentence Classification](http://arxiv.org/pdf/1408.5882v2.pdf)" by Yoon Kim)
```
# CNN + LSTM

model_CNN_LSTM = Sequential()
model_CNN_LSTM.add(embedding_layer)
model_CNN_LSTM.add(Dropout(0.5))
model_CNN_LSTM.add(Convolution1D(filters=nb_filter,
                        kernel_size=filter_length,
                        border_mode=border_mode,
                        activation=cnn_activation,
                        subsample_length=1))
model_CNN_LSTM.add(MaxPooling1D(pool_size=pool_length))
model_CNN_LSTM.add(LSTM(units=output_size,activation=rnn_activation,recurrent_activation=recurrent_activation))
model_CNN_LSTM.add(Dropout(0.25))
model_CNN_LSTM.add(Dense(1))
model_CNN_LSTM.add(Activation('sigmoid'))
model_CNN_LSTM.compile(loss=loss,
                       optimizer=optimizer,
                       metrics=['accuracy'])

print('CNN + LSTM')

from keras.callbacks import EarlyStopping, TensorBoard, ModelCheckpoint

path = 'weights.{epoch:02d}-{loss:.2f}-{acc:.2f}.hdf5'
model_checkpoint = ModelCheckpoint(path, monitor = 'loss', verbose = 1, save_best_only = True, mode = 'auto')
early_stopping = EarlyStopping(monitor='loss', patience = 8, verbose = 1, mode = 'auto')

history_CNN_LSTM = model_CNN_LSTM.fit(x, y, batch_size = batch_size, 
                                      epochs = nb_epoch, 
                                      validation_split = validation_split, 
                                      shuffle = shuffle, 
                                      verbose = 1, 
                                      callbacks = [model_checkpoint, early_stopping])
```

7. Define two plot function to plot the history of accuracy and loss by using matplotlib.
```
import matplotlib.pyplot as plt
plt.style.use('ggplot')

def plot_history_ggplot(history):
    acc = history.history['acc']
    val_acc = history.history['val_acc']
    loss = history.history['loss']
    val_loss = history.history['val_loss']
    x = range(1, len(acc) + 1)

    plt.figure(figsize=(12, 5))
    plt.subplot(1, 2, 1)
    plt.plot(x, acc, 'b', label='Training acc')
    plt.plot(x, val_acc, 'r', label='Validation acc')
    plt.title('Training and validation accuracy')
    plt.legend()
    plt.subplot(1, 2, 2)
    plt.plot(x, loss, 'b', label='Training loss')
    plt.plot(x, val_loss, 'r', label='Validation loss')
    plt.title('Training and validation loss')
    plt.legend()
    
def plot_history(history):
    # plot results
    loss = history.history['loss']
    val_loss = history.history['val_loss']

    acc = history.history['acc']
    val_acc = history.history['val_acc']

    plt.figure(figsize=(10,10))
    plt.subplot(2,1,1)
    plt.title('Loss')
    epochs = len(loss)
    plt.plot(range(epochs), loss, marker='.', label='loss')
    plt.plot(range(epochs), val_loss, marker='.', label='val_loss')
    plt.legend(loc='best')
    ax = plt.gca()
    ax.spines['bottom'].set_linewidth(5)
    ax.spines['top'].set_linewidth(5)
    ax.spines['right'].set_linewidth(5)
    ax.spines['left'].set_linewidth(5)
    ax.set_facecolor('snow')
    plt.grid(color='lightgray', linestyle='-', linewidth=1)
    plt.xlabel('epoch')
    plt.ylabel('acc')

    plt.subplot(2,1,2)
    plt.title('Accuracy')
    plt.plot(range(epochs), acc, marker='.', label='acc')
    plt.plot(range(epochs), val_acc, marker='.', label='val_acc')
    plt.legend(loc='best')
    ax = plt.gca()
    ax.spines['bottom'].set_linewidth(5)
    ax.spines['top'].set_linewidth(5)
    ax.spines['right'].set_linewidth(5)
    ax.spines['left'].set_linewidth(5)
    ax.set_facecolor('snow')
    plt.grid(color='lightgray', linestyle='-', linewidth=1)
    plt.xlabel('epoch')
    plt.ylabel('acc')
    plt.show()
```

8. Compare the performance among the five deep learning models.
* Simple RNN
![Simple RNN](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%884.47.22.png)

* GRU
![GRU](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%884.47.42.png)

* LSTM
![LSTM](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%884.48.14.png)

* BiLSTM
![BiLSTM](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%884.48.32.png)

* CNN + LSTM
![CNN + LSTM](https://github.com/penguinwang96825/Text_Classification/blob/master/image/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%202019-05-08%20%E4%B8%8B%E5%8D%884.48.45.png)

9. In training a neural network, f1 score is an important metric to evaluate the performance of classification models, especially for unbalanced classes where the binary accuracy is useless.
```
tk = text.Tokenizer(nb_words=max_features, filters='!"#$%&()*+,-./:;<=>?@[\\]^_`{|}~\t\n', lower=True, split=" ")
tk.fit_on_texts(sentences_test)
sentences_test = tk.texts_to_sequences(sentences_test)
word_index = tk.word_index
sentences_test = sequence.pad_sequences(sentences_test,maxlen=max_len)

y_pred_temp = model_CNN_LSTM.predict_classes(sentences_test).tolist()
y_pred = []
for i in y_pred_temp:
    y_pred.append(str(i).strip('[]'))
y_pred = [int(i) for i in y_pred]
print("Size of label: ", len(y_pred))

y_test_temp = y_test.tolist()
y_test = []
for i in y_test_temp:
    y_test.append(str(i).strip('[]'))
y_test = [int(i) for i in y_test]
print("Size of label: ", len(y_test))
```
```
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score

# accuracy: (tp + tn) / (p + n)
accuracy = accuracy_score(y_test, y_pred)
print('Accuracy: %f' % accuracy)
# precision tp / (tp + fp)
precision = precision_score(y_test, y_pred)
print('Precision: %f' % precision)
# recall: tp / (tp + fn)
recall = recall_score(y_test, y_pred)
print('Recall: %f' % recall)
# f1: 2 tp / (2 tp + fp + fn)
f1 = f1_score(y_test, y_pred)
print('F1 score: %f' % f1)
```
