# Personalization

With SMTPeter you can easily personalize your mailings, when you use the
[REST API](rest-send) to send your mails. You simply add an extra property,
"data", to your JSON, or add extra information to the property "recipients"
if you send a mass mailing. This information can be used to replace parts
of your mail.

## Add personal data to the JSON

If you send a mail with only one recipient, you can add the personal data
with property "data".

```json
{
    "recipient": "john@doe.com",
    "data"     : {
        "firstname"  : "John",
        "familyname" : "Doe"
    }
}
```
If you have multiple recipients, the personal data can be passed as follows:

```json
{   
    "recipients" : [
        "jane@doe.com": {
            "firsname": "Jane",
            "familyname": "Doe"
        },
        "john@doe.com": {
            "firstname": "John",
            "familyname": "Doe"
        }
    ]
}
```
When you use the above JSONs you can access the content of "firstname" and
"familyname" in the "from" and "to" address, the header, the text, and the
html fields or in the "mime". To make life easy for you, you standard have
access to the "envelope" and "recipient" information.

## Using the personal data

Let's say you have the following JSON:
```json
{
    "recipient": "john@example.com",
    "data": {
        "ourname": "The SMTPeter test team",
        "name": "John Doe",
        "age": 33,
        "job": "programmer",
        "children": [
            "Peter", "Angela", "Brandon"
        ]
    }
    "from": "....",
    "to": "...",
    ...
}
```

If you use the above JSON data for your mail, you can use inside the "from"
and "to" address, the subject line, and inside the text and HTML versions
of your email these variables.

```json
{
    "recipient": ...,
    "data": ...,
    "from": "info <info@example.com>",
    "to": "{$name} <john@example.org>",
    "subject": "Hello {$name}!",
    "text": "Hi {$name},\n\nYour age is {$age}, and your job is {$job}.\n\nCheers,\n\n{$ourname}"
````
If you had used the above JSON, SMTPeter would replace the variables
in the "from", and "to" address, the subject line and the text version. 
For ease of use the "envelope" and "recipient" are already extracted from
the mail for you. You can use these without specifying them as a data
property.

Note that with this personalization, you can also send your mass mails with
only one REST call. 


## Simple programming

The syntax for the personalization variables are loosely based on the 
Smarty template engine that is used in many PHP projects. Variables
use the {$name} syntax, and you can even use simple programming statements:

````text
{if $age > 30}
    Text that is shown to everyone older than 30
{else}
    Text visible for others
{/if}

{foreach $children as $child}
    One of your children is named {$child}.
{/foreach}
````

However, templates can not be used for real programming. The supported
programming constructs are relatively simple and are more or less restricted 
to {if} and {foreach} statements. 


## Variable modifiers

You can pass data to variable modifiers. If you use a variable, you can
pass it through a modifier to modify the data.

````text
Hello {$name|escape},

Your name {$name|escape} is {$name|strlen} characters long.

Bye!
````

The following table lists all supported modifiers:

<table>
    <tr>
        <td>base64encode</td>
        <td>base64 encoder</td>
    </tr>
    <tr>
        <td>base64decode</td>
        <td>base64 decoder</td>
    </tr>
    <tr>
        <td>cat:"string"</td>
        <td>concatenates string to variable</td>
    </tr>
    <tr>
        <td>count</td>
        <td>count number of elements in variable</td>
    </tr>
    <tr>
        <td>count_characters</td>
        <td>count number of characters in a string</td>
    </tr>
    <tr>
        <td>count_paragraphs</td>
        <td>count number of paragraphs in a text (by counting newlines)</td>
    </tr>
    <tr>
        <td>count_words</td>
        <td>count number of words in a text</td>
    </tr>
    <tr>
        <td>default:default value</td>
        <td>use default value if variable is not set</td>
    </tr>
    <tr>
        <td>empty</td>
        <td>check whether a variable is empty</td>
    </tr>
    <tr>
        <td>escape:"string"</td>
        <td>escape html characters (or other chars) inside a string</td>
    </tr>
    <tr>
        <td>indent:num = 1:char = " "</td>
        <td>put num whitespaces in front of every line</td>
    </tr>
    <tr>
        <td>md5</td>
        <td>perform md5 hashing</td>
    </tr>
    <tr>
        <td>nl2br</td>
        <td>replace newlines with html br tags</td>
    </tr>
    <tr>
        <td>range:start = 0:end</td>
        <td>truncate list to get the items between positions start and end</td>
    </tr>
    <tr>
        <td>regex_replace:regex:replace_text</td>
        <td>replace substrings using regular expression</td>
    </tr>
    <tr>
        <td>replace:"string1":"string2"</td>
        <td>replace occurrences of string1 with string2</td>
    </tr>
    <tr>
        <td>sha1</td>
        <td>perform sha1 hashing</td>
    </tr>
    <tr>
        <td>sha256</td>
        <td>perform sha256 hashing</td>
    </tr>
    <tr>
        <td>sha512</td>
        <td>sha512 hashing</td>
    </tr>
    <tr>
        <td>spacify:separator = " "</td>
        <td>place a separator between every input character</td>
    </tr>
    <tr>
        <td>strlen</td>
        <td>count the characters in a string</td>
    </tr>
    <tr>
        <td>strstr:"substring":before = false</td>
        <td>return the string starting from the first occurrence of substring if before = false. otherwise return the string until the first occurrence.</td>
    </tr>
    <tr>
        <td>substr:start position:length</td>
        <td>return the substring from start position onward, optionally truncated after length characters</td>
    </tr>
    <tr>
        <td>tolower</td>
        <td>convert all characters to lower case</td>
    </tr>
    <tr>
        <td>toupper</td>
        <td>convert all characters to upper case</td>
    </tr>
    <tr>
        <td>trim:characters = " \t\n\r\0\x0B"</td>
        <td>trim the specified characters off both sides of the input</td>
    </tr>
    <tr>
        <td>truncate:length = 80:etc = "...":break_words = false</td>
        <td>truncate inputs that are longer than length and append etc at the end. break_words = true allows truncating parts of words</td>
    </tr>
    <tr>
        <td>ucfirst</td>
        <td>replace first character with an upper case character</td>
    </tr>
    <tr>
        <td>urlencode</td>
        <td>encode input for use in an url</td>
    </tr>
    <tr>
        <td>urldecode</td>
        <td>decode input for use in an url</td>
    </tr>
</table>