---
type: posts
title: Configure MySQL for Arduino in Python
author: Amutheezan Sivagnanam
category: Tech Issues
date: 2016-09-11
last_modified_at: 2016-09-22
tags:
- arduino
- python

---

This guide is only applicable for Windows.

This article explains the steps associated with configuring ```MySQL``` for Arduino in Python.

### Steps

1. First install ```XAMPP```/```WAMP```. Then open its control panel and start ```Apache``` and ```MySQL``` servers.
   After that, click the admin button and goto ```PHPMyAdmin``` and create the database to store the data.

2. Install Python 2.7, and install ```pyserial```, ```MYSQLdb``` libraries.

3. Install ```pyserial```, you can also download it
   from [here](https://pypi.python.org/packages/47/c9/7802e11ab388ad1539de716649add8bb8ca8bdff660364b3a404f79c27b7/pyserial-2.7.win32.exe)[https://pypi.python.org/pypi/pyserial/2.7#downloads](https://pypi.python.org/pypi/pyserial/2.7#downloads)[.](https://pypi.python.org/pypi/pyserial/2.7)
   Download the ".exe" file and just run it.

4. Install ```MYSQLdb```, you can also download it
   from [https://pypi.python.org/pypi/MySQL-python/1.2.5#downloads](https://pypi.python.org/pypi/MySQL-python/1.2.5#downloads).
   Download the ".exe" file and just run it.

5. Then, create the required python program to update the database. For this, you need to add two
   import ```import serial``` and ```import MysqlDb```.

6. Install Arduino and write the program to read values from the sensor and print the values
   using ```serial.Println(readData)```.

I referred to this code in [4](https://github.com/surendharreddy/Arduino-MySQL) for my project; I hope this will help
you.

### References

1. [http://stackoverflow.com/questions/28335859/importerror-no-module-named-serial-in-windows-7-python-2-7-and-python-3-3](http://stackoverflow.com/questions/28335859/importerror-no-module-named-serial-in-windows-7-python-2-7-and-python-3-3)
2. [http://stackoverflow.com/questions/8491111/pyserial-for-python-2-7-2](http://stackoverflow.com/questions/8491111/pyserial-for-python-2-7-2)
3. [http://www.instructables.com/id/Interface-Arduino-to-MySQL-using-Python/step4/Python-TIEM/](http://www.instructables.com/id/Interface-Arduino-to-MySQL-using-Python/step4/Python-TIEM/)
4. [https://github.com/surendharreddy/Arduino-MySQL](https://github.com/surendharreddy/Arduino-MySQL)
