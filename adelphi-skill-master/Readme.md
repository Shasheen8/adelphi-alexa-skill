# Adelphi University Academic Calendar Alexa Skill

## Installation Instructions

**adelphi_skill.py** requires python 3.6 (or later)

Prior to uploading a skill to AWS you first need to set up your [credentials file](https://aws.amazon.com/blogs/security/a-new-and-standardized-way-to-manage-credentials-in-the-aws-sdks/).


In order to start the skill first create a virtual environment in which to run the code.

```
virtualenv env
source env/bin/activate

pip install -r requirements.txt
```


After that setup zappa to aid in uploading to AWS Lambda

```
zappa init
zappa deploy dev
```
