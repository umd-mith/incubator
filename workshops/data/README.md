Data workshop
=============

Unix text tools overview
------------------------

I highly recommend spending some time with Kenneth Ward Church's classic
[Unix for Poets](http://ufal.mff.cuni.cz/~hladka/tutorial/UnixforPoets.pdf).

Today we'll be using the following:

 * `grep`
 * `tr`
 * `sed`

And the following, but you don't need to worry about them as much for now:

 * `cut`
 * `sort`
 * `uniq`

Named Entity Recognition
------------------------

First download the [Apache OpenNLP toolkit](http://opennlp.apache.org/):

    mkdir tools
    cd tools
    curl -O http://mirrors.ibiblio.org/apache//incubator/opennlp/apache-opennlp-1.5.2-incubating-bin.tar.gz
    tar zxvf apache-opennlp-1.5.2-incubating-bin.tar.gz

Next download [the models](http://opennlp.sourceforge.net/models-1.5/):

    cd ..
    mkdir models
    cd models
    curl -O http://opennlp.sourceforge.net/models-1.5/en-token.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-sent.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-pos-maxent.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-ner-date.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-ner-location.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-ner-money.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-ner-organization.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-ner-person.bin
    curl -O http://opennlp.sourceforge.net/models-1.5/en-ner-time.bin

I've only listed the English-language models here, but there are also models
for many components for other languages.

I'll download several of the works of George Berkeley as examples:

    cd ../texts
    curl -A "Mozilla/4.0" -O http://www.gutenberg.org/cache/epub/4722/pg4722.txt
    curl -A "Mozilla/4.0" -O http://www.gutenberg.org/cache/epub/31848/pg31848.txt
    curl -A "Mozilla/4.0" -O http://www.gutenberg.org/cache/epub/4543/pg4543.txt
    curl -A "Mozilla/4.0" -O http://www.gutenberg.org/cache/epub/4724/pg4724.txt
    curl -A "Mozilla/4.0" -O http://www.gutenberg.org/cache/epub/4723/pg4723.txt

We have to specify the user agent since Project Gutenberg blocks `curl` (and `wget`) by default.
They [say](http://www.gutenberg.org/error403.php),
"The Project Gutenberg Web Site is for human (non-automated) users only",
but we are human users (just ones who are using an alternative browser), so I'm going
to assume this is kosher. Just don't do it in large batches.

We'll start with "A Proposal for the Better Supplying of Churches in Our Foreign Plantations".
We need to strip the Gutenberg footer:

    head -n 780 pg31848.txt > berkeley-01.txt

Now we can break into sentences and tokenize:

    cd ..
    apache-opennlp-1.5.2-incubating/bin/opennlp \
      SentenceDetector models/en-sent.bin < texts/berkeley-01.txt | \
      apache-opennlp-1.5.2-incubating/bin/opennlp \
      TokenizerME models/en-token.bin > texts/berkeley-01-tokenized.txt

Now we can find person names:

    apache-opennlp-1.5.2-incubating/bin/opennlp \
      TokenNameFinder models/en-ner-person.bin \
      < texts/berkeley-01-tokenized.txt > people-01.txt

Now we have a bit of a problem, since the output looks like this:

    <START:person> Clement Cotterel Bart <END> . in Dover-street .

I.e, roughly XML-ish markup that isn't actually XML. We can't just use `grep` straightforwardly,
since there may be multiple entities in a line, or entities that span line breaks. So we first use
`tr` to turn line breaks into spaces, and `sed` to insert new line breaks after entities:

    tr '\n' ' ' < people-01.txt | sed 's/<END>/<END>\n/g' > people-02.txt

Now we can `grep`:

    egrep -o '<START:person>[^<]+' people-02.txt | \
      cut -f 2 -d '>' | cut -f 1 -d '<' | grep -v '_'

Which gives us this:

    George Berkeley 
    George Berkeley 
    Christian 
    Christian 
    King James I. 
    George Berkeley 
    Dean 
    William Thompson 
    Jonathan Rogers 
    James King 
    Lord Bishop 
    John Arbuthnot M.D. 
    Martin Benson 
    Alderman 
    Clement Cotterel Bart 
    Thomas Crosse Kt 
    Daniel Dolins Kt 
    Thomas Green Esq 
    Edward Harley Esq 
    Henry Hoare Esquires 
    Lord Chancellor 
    Dr. Moss 
    Dean 
    Dr. Pierce 
    Dean 
    William Wentworth Bart 
    William Lord Arch-Bishop 
    Peter Lord King 
    Thomas Duke 
    Lord Bishop 

Not perfect, but not too bad for a quick experiment. We can find place names similarly:

    apache-opennlp-1.5.2-incubating/bin/opennlp \
      TokenNameFinder models/en-ner-location.bin < texts/berkeley-01-tokenized.txt | \
      tr '\n' ' ' | sed 's/<END>/<END>\n/g' | \
      egrep -o '<START:location>[^<]+' | cut -f 2 -d '>' | cut -f 1 -d '<' | \
      sort | uniq -c | sort -n

Note that I've combined all the steps into a single pipeline and have grouped and counted
the results.

Topic Modeling
--------------

Next we'll download and unzip [MALLET](http://mallet.cs.umass.edu/):

    curl -O http://mallet.cs.umass.edu/dist/mallet-2.0.6.tar.gz
    tar zxvf mallet-2.0.6.tar.gz

Now we import these into MALLET's binary format:

    mallet-2.0.6/bin/mallet import-dir \
      --input texts/[1-2]* --output mh.mallet \
      --keep-sequence  --remove-stopwords

And now we can learn a model:

    mallet-2.0.6/bin/mallet train-topics --num-threads 2 \
      --input mh.mallet --num-topics 30 --optimize-interval 10 \
      --output-topic-keys mh-30-keys.txt --output-model mh-30.model



