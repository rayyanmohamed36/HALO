# Quick Notes

Methods of Positioning
---

I want to use multiple methods layered on top of each other in order to reduce false positives. I want to make it as accurate as possible. So far I have 3 different methods of indoor positioning in mind:
- BLE via ESP32 or BLE modules
- 2.4GHz Radio via nRF24L01 module
- Ultrasonic model via transmitters and receivers

Considerations
---
However I also do not want to make the conditions to flag the wearable being indoors too strict as this may result in false negatives and would reduce the accuracy of the product. As the goal is to maximise the accuracy of the method, testing should be conducted within a closed room in order to estimate the best possible commbination of methods. Furthermore, the environment will also play a role in the accuracy of the model. A room that has walls made of cement may have higher accuracy as it reduces the signal strength by a much larger magnitude than a room which has walls made of drywall.

Price is also a factor which affects the methods which are used, a major method which I overlooked was the use of UWB technology, the only reason this model wasn't planned out was due to it's high cost. One of the goals of HALO is to have a low price in order to maximise the benefit which is gained by the users. Of course, if UWB technology was the most accurate and only model which could be used to meet our goals it would have been used, still the same goal can be achieved through a bit of research and the correct combination of factors.

Reasons for BLE
---

The main reason for choosing BLE as the first method that I researched was due to my previous research of BLE positioning in smart walking cane applications. I developed a method of estimating distance using triangulation and RSSI calculation using esp32s as beacons and an esp32 as a receiver. As I developed this model during an internship I never got around to testing the model and optimizing its accuracy.




