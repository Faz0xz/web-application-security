# Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected XSS  
**Difficulty:** Practitioner  

## Objective

This lab contains a reflected cross-site scripting vulnerability in the search query tracking functionality where angle brackets and double are HTML encoded and single quotes are escaped. To solve this lab, perform a cross-site scripting attack that breaks out of the JavaScript string and calls the alert function. 

## Initial Testing

Input : Test<>"\' 
Output : 'Test<>"\\''

Input is being wrapped in single quotes at the beginning and end. Backslash is also being escaped by adding another backslash, yet I expected the angle brackets to be html-encoded yet they are not, maybe it's because the input is being treated as a singular string. Further testing is required.

<img width="776" height="106" alt="image" src="https://github.com/user-attachments/assets/14e70637-830e-4cc3-b5c7-a8612d05d59e" />

<img width="347" height="35" alt="image" src="https://github.com/user-attachments/assets/32b33f7b-fffe-4966-b935-1fd3da736697" />

When inspecting the page via the dev console I also noticed an inline javascript block. 
<img width="827" height="67" alt="image" src="https://github.com/user-attachments/assets/0c2581a6-8a3e-49eb-af01-d142bf491cf3" />

 My input is being html encoded when it is given to the js function where it is processed.
