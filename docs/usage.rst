Usage
=====

Basic Usage
-----------

Grab some HTML:

    >>> import requests
    >>> html = requests.get('https://www.github.com/').text

Then use :func:`formasaurus.extract_forms <formasaurus.classifiers.extract_forms>`
to detect form and field types:

    >>> import formasaurus
    >>> formasaurus.extract_forms(html)
    [(<Element form at 0x1150ba0e8>,
      {'fields': {'q': 'search query'}, 'form': 'search'}),
     (<Element form at 0x1150ba138>,
      {'fields': {'user[email]': 'email',
        'user[login]': 'username',
        'user[password]': 'password'},
       'form': 'registration'})]

.. note::

    To detect form and field types Formasaurus needs to train prediction
    models on user machine. This is done automatically on first call;
    models are saved to a file and then reused.

:func:`formasaurus.extract_forms <formasaurus.classifiers.extract_forms>`
returns a list of (form, info) tuples, one tuple for each ``<form>``
element on a page. ``form`` is a lxml Element for a form,
``info`` dict contains form and field types.

Only fields which are

1. visible to user;
2. have non-empty ``name`` attribute

are returned - other fields usually should be either submitted as-is
(hidden fields) or not sent to the server at all (fields without
``name`` attribute).

There are edge cases like fields filled with JS or fields which are made
invisible using CSS, but all bets are off if page uses JS heavily and all
we have is HTML source.

By default, info dict contains only most likely form and field types.
To get probabilities pass ``proba=True``:

    >>> formasaurus.extract_forms(html, proba=True, threshold=0.05)
    [(<Element form at 0x1150db408>,
      {'fields': {'q': {'search query': 0.999129068423436}},
       'form': {'search': 0.99580680143321776}}),
     (<Element form at 0x1150dbae8>,
      {'fields': {'user[email]': {'email': 0.9980438256540791},
        'user[login]': {'username': 0.9877912041558733},
        'user[password]': {'password': 0.9968113886622333}},
       'form': {'login': 0.12481875549840604,
        'registration': 0.86248202363754578}})]

Note that Formasaurus is less certain about the second form type - it thinks
most likely the form is a registration form (0.86%), but the form
also looks similar to a login form (12%).

``threshold`` argument can be used to filter out low-probability options;
we used 0.05 in this example. To get probabilities of all classes use
``threshold=0``.

To classify individual forms use
:func:`formasaurus.classify <formasaurus.classifiers.classify>`
or :func:`formasaurus.classify_proba <formasaurus.classifiers.classify_proba>`.
They accept lxml <form> Elements. Let's load an HTML file using lxml:

    >>> import lxml.html
    >>> tree = lxml.html.parse("http://google.com")

and then classify the first form on this page:

    >>> form = tree.xpath('//form')[0]
    >>> formasaurus.classify(form)
    {'fields': {'btnG': 'submit button',
      'btnI': 'submit button',
      'q': 'search query'},
     'form': 'search'}

    >>> formasaurus.classify_proba(form, threshold=0.1)
    {'fields': {'btnG': {'submit button': 0.9254039698573596},
      'btnI': {'submit button': 0.9642014575642849},
      'q': {'search query': 0.9959819637966439}},
     'form': {'search': 0.98794025545508202}}

In this example the data is loaded from an URL; of course, data may be
loaded from a local file or from an in-memory object, or you may already
have the tree loaded (e.g. with Scrapy).

Form Types
----------

Formasaurus detects these form types::

                             precision    recall  f1-score   support

                     search       0.91      0.96      0.94       364
                      login       0.96      0.96      0.96       221
               registration       0.97      0.86      0.91       153
    password/login recovery       0.88      0.88      0.88        95
            contact/comment       0.87      0.93      0.90       120
          join mailing list       0.90      0.89      0.90       107
          order/add to cart       0.95      0.66      0.78        62
                      other       0.67      0.70      0.69       122

                avg / total       0.90      0.90      0.89      1244

    89.5% forms are classified correctly.

Quality is estimated based on cross-validation results:
all annotated data is split into 20 folds, then model is trained on 19 folds
and tries to predict form types in the remaining fold. This is repeated to get
predictions for the whole dataset.

See also: https://en.wikipedia.org/wiki/Precision_and_recall

Field Types
-----------

By deafult, Formasaurus detects these field types:

* ``username``
* ``password``
* ``password confirmation`` - "enter the same password again"
* ``email``
* ``email confirmation`` - "enter the same email again"
* ``username or email`` - a field where both username and email are accepted
* ``captcha`` - image captcha or a puzzle to solve
* ``honeypot`` - this field usually should be left blank
* ``TOS confirmation`` - "I agree with Terms of Service",
  "I agree to follow website rules", "It is OK to process my personal info", etc.
* ``receive emails confirmation`` - a checkbox which means
  "yes, it is ok to send me some sort of emails"
* ``remember me checkbox`` - common on login forms
* ``submit button`` - a button user should click to submit this form
* ``cancel button``
* ``reset/clear button``
* ``first name``
* ``last name``
* ``middle name``
* ``full name``
* ``organization name``
* ``gender``
* ``day``
* ``month``
* ``year``
* ``full date``
* ``time zone``
* ``DST`` - Daylight saving time preference
* ``country``
* ``city``
* ``state``
* ``address`` - other address information
* ``postal code``
* ``phone`` - phone number or its part
* ``fax``
* ``url``
* ``OpenID``
* ``about me text``
* ``comment text``
* ``comment title or subject``
* ``security question`` - "mother's maiden name"
* ``answer to security question``
* ``search query``
* ``search category / refinement`` - search parameter, filtering option
* ``product quantity``
* ``style select`` - style/theme select, common on forums
* ``sorting option`` - asc/desc order, items per page
* ``other number``
* ``other read-only`` - field with information user shouldn't change
* all other fields are classified as ``other``.

Quality estimates (based on 20-fold cross-validation)::

                                  precision    recall  f1-score   support

                        username       0.81      0.91      0.85       187
                        password       0.99      0.99      0.99       338
           password confirmation       0.96      0.99      0.97        97
                           email       0.94      0.97      0.95       544
              email confirmation       0.96      0.85      0.90        26
               username or email       0.82      0.41      0.55        34
                         captcha       0.84      0.82      0.83        83
                        honeypot       0.17      0.06      0.08        18
                TOS confirmation       0.81      0.50      0.62        84
     receive emails confirmation       0.36      0.59      0.45        83
            remember me checkbox       0.94      1.00      0.97       117
                   submit button       0.96      0.97      0.96       334
                   cancel button       0.86      0.60      0.71        10
              reset/clear button       1.00      0.83      0.91        12
                      first name       0.92      0.86      0.89        95
                       last name       0.88      0.85      0.86        93
                     middle name       1.00      0.67      0.80         6
                       full name       0.74      0.82      0.78       120
               organization name       0.81      0.43      0.57        30
                          gender       0.98      0.80      0.88        75
                       time zone       1.00      0.71      0.83         7
                             DST       1.00      1.00      1.00         5
                         country       0.85      0.72      0.78        47
                            city       0.95      0.68      0.79        53
                           state       1.00      0.63      0.77        38
                         address       0.75      0.64      0.69        84
                     postal code       0.95      0.79      0.87        78
                           phone       0.83      0.85      0.84       102
                             fax       1.00      1.00      1.00         8
                             url       0.88      0.66      0.75        32
                          OpenID       1.00      0.75      0.86         4
                   about me text       0.50      0.33      0.40        12
                    comment text       0.86      0.93      0.89       121
        comment title or subject       0.67      0.45      0.53       121
               security question       1.00      0.44      0.62         9
     answer to security question       0.80      0.57      0.67         7
                    search query       0.89      0.95      0.92       350
    search category / refinement       0.91      0.87      0.89       376
                product quantity       0.98      0.84      0.90        55
                    style select       0.93      1.00      0.97        14
                  sorting option       0.87      0.50      0.63        26
                    other number       0.27      0.15      0.19        27
                       full date       0.47      0.35      0.40        20
                             day       0.96      0.88      0.92        25
                           month       0.96      0.89      0.92        27
                            year       0.97      0.88      0.92        34
                 other read-only       1.00      0.42      0.59        24
                           other       0.65      0.78      0.71       710

                     avg / total       0.85      0.84      0.83      4802

    83.7% fields are classified correctly.
    All fields are classified correctly in 75.3% forms.
