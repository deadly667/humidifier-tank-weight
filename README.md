# Humidifier tank weight sensor

<img src="/content/20201218_223717.jpg" width="350" >

## Motivation

I have Rowenta HU5220 air humidifier which is pretty good but it doesn't have any smart features. It has water tank capacity of 5.9 l and I'm living in old apartment in which humidity is normally very low, about 20 % during winter. So my humidifier works all the time and usually 5 l of water lasts for 24 hours to keep humidity in 50 % window. What usually happens is that I don't notice that water is gone and humidifier turns himself off and humidity drops. I was thinking how I can make my humidifier smart so that he can warn me if water level is low so that I can add water. After some research I decided that I will go with weight sensors with HX711 and D1 mini with ESPHome connected to Home Assistant. 

Idea is that we will create digital scale which will measure remaining water in a humidifier and warn us if remaining water weight drops under 10 %. 

## Ingredients

For this project you will need:

* Load cells ([Link](https://www.aliexpress.com/item/1005001433083364.html))
* HX711 module ([Link](https://www.aliexpress.com/item/32961313251.html))
* D1 mini ([Link](https://www.aliexpress.com/item/32952854730.html))
* Dupont cables
* Wood board


## Wiring 

Here you can see wiring diagram:

![Wiring](/content/humidity_tank_wiring_bb.png)

![Wiring Schema](/content/humidity_tank_wiring_schem.png)


First you need wood board. I cut mine to 20 cm x 20 cm, but you will cut yours to dimensions which your humidifier needs.

**(Optional)** I also 3D printed Load sensor holders ([Link](https://www.thingiverse.com/thing:2486831)), but you can just drill small space in the wood board so that Load sensor has space to move. 

Glue Load sensors to the wood board or 3D printed holders.

After you glue it down, advice is that you mark them which is upper left/right and which is lower left/right. It will be easier to connect everything. 

<img src="/content/20201218_125119.jpg" width="350" >

<img src="/content/20201218_172659.jpg" width="350" >


I just glue all wires with hot glue. It doesn't need to be pretty, it will be under the board so nobody will see that. 

<img src="/content/20201218_183959.jpg" width="350" >

Connect Load wires with HX711 module and HX711 with D1 mini and that's it. 

## ESPHome

Now we need to flash our D1 mini with ESPHome. If you still didn't use ESPHome I recommend that you take a look into docs [ESPHome docs](https://esphome.io/index.html). Also there is a lot of ESPHOME HowTos videos on YouTube (e.g.  [BeardedTinker](https://www.youtube.com/watch?v=XMFn8fKhUFA)).

Before initial flashing of *humidity_tank_weight.yml* to D1 mini you need to set your WiFi SSID and Password and also put some passwords to *ap*, *api* and *ota* part. Change timezone and that's it for initial flash. Flash D1 mini and check that D1 mini connects to your WiFi.

### Calibration

To make that weight is properly measured we need to initialize it. First you want to measure humidifier without water. Look in the logs of ESPHome and you will see something like this:

```
[18:52:44][D][hx711:031]: 'Humidity tank weight': Got value -711989
[18:52:44][D][sensor:092]: 'Humidity tank weight': Sending state 3655.23633 g with 0 decimals of accuracy
[18:52:50][D][sensor:092]: 'Humidity tank WiFi Signal': Sending state -58.00000 dB with 0 decimals of accuracy
[18:52:52][D][sensor:092]: 'Humidity tank Uptime': Sending state 87473.17969 s with 0 decimals of accuracy
```

the important line is this `[18:52:44][D][hx711:031]: 'Humidity tank weight': Got value -711989`

Write number which you got and this will be our weight when we have 0 g of water.

Next step is to fill humidifier with water. Fill your humidifier till max. Here we need to know how much of water you put there. The easiest way is if you have body scale or kitchen scale, and you put our digital scale with humidifier on top of it. Push tare button so that we have 0 with everything on top and then fill with water and read the value. Write that value down and again go to ESPHome logs and read which value has our digital scale measured. 

Now with this 3 values we need to edit our code.


First value when humidifier was empty goes to first line of *calibrate_linear* part. For me, it was -628112. Be careful with sign!  
Second value which we read from ESPHome logs when humidifier was full we put in second row. For me, it was -754321.  
And finally last value of how many grams of water we put in humidifier is after arrow in second line. For me, it was 5500 grams. 

```
...

 filters:
      - calibrate_linear:
          - -628112 -> 0 # empty Humidifier
          - -754321 -> 5500 # fully loaded Humidifier
...
```


We are almost done. Only one thing that is left is to set 10 % limit. In binary_sensor part `} else if (id(humidity_tank_weight).state < 500) {` you need to put value in grams so that ESPHome knows when the level of remaining water is under 10 %. I put 500 grams because usually I only fills 5 l of water in my humidifier. 


```
...
binary_sensor:
  - platform: template
    name: "Humidity tank < 10%"
    filters:
      - delayed_off: 15s
    lambda: |-
      if (isnan(id(humidity_tank_weight).state)) {
        return {};
      } else if (id(humidity_tank_weight).state < 500) {
        // Running
        return true;
      } else {
        // Not running
        return false;
      }
...
```

Now we have to save that file and upload code to our D1 mini. That's it!

## Home Assistant

If you have ESPHome integration in Home assistant already installed it will recognize your new ESPHome device and ask you if you want to add it. 

![Home Assistant device](/content/HA.PNG)

Also, you need to put some simple automation if you want water level notification to telegram. Trigger is  binary_sensor state on.

![Telegram notification](/content/telegram.jpg)




## Credits:

* Idea comes from: https://vigonotion.com/blog/monitor-remainding-water-bottles/
* Load sensor holders: https://www.thingiverse.com/thing:2486831

