Operating System (OS): Windows

Steps to Install Java on Windows: 
1. Go to [https://www.oracle.com/hk/java/technologies/downloads/](https://www.oracle.com/hk/java/technologies/downloads/#jdk24-windows).
2. Download `.exe` file.
3. Click the downloaded `.exe` file.
4. Click "Next".
![image](https://github.com/user-attachments/assets/99484773-7e42-4412-ab43-bd4f9d7f39f7)
5. Click "Change" to change the installation folder.
6. Click "Next".
7. Click "Close".
![image](https://github.com/user-attachments/assets/b73af54f-4154-4b72-8d45-7d7858aec601)

To check if Java has successfully been installed, type `java -version` into the Command Prompt.

Steps to Write a Simple Java Program in a Text Editor: 
1. Open a text editor (e.g., Notepad++).
2. Copy and paste the following code:
```
// Simple Java program
public class DarkTealCoder
{
    public static void main(String[] args)
    {
        System.out.println("Hello, World!");
    }
}
```
3. Save it as "DarkTealCoder.java".
4. Open the Command Prompt or PowerShell.
5. Use the `cd` command to navigate to the `.java` file location.
6. Type `javac DarkTealCoder.java` to compile the `.java` file into a `.class` file.
7. Type `java DarkTealCoder` to run the program (without `.class`).
