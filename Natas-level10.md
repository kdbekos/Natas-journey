Challenge: Natas Level 10 → 11

1. Objective
Find the password for Natas 11, located at /etc/natas_webpass/natas11.

2. Analysis
The application provides a search interface that queries a dictionary file using the grep command. Unlike the previous level (Natas 9), this level implements a character blacklist via a regular expression:
if($key != "") {
    if(preg_match('/[;|&]/',$key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i $key dictionary.txt");
    }

3. Exploitation Strategy

The vulnerability lies in the fact that the input $key is passed directly into a shell command without being escaped. While we cannot chain a new command, we can still manipulate the arguments of the existing grep command.

The structure of the command is:
grep -i [USER_INPUT] dictionary.txt

By including a space in our input, we can trick grep into treating our input as both a pattern and an additional file path.
