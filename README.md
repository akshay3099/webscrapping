#!/usr/bin/env python
# coding: utf-8
#Importing required libraries and modules
import pandas as pd
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import NoSuchElementException
from geopy.geocoders import Nominatim
from geopy import geocoders
import folium
from folium.plugins import MarkerCluster
import time

# # Setting up chrome web driver
s=Service("C:\Program Files (x86)\chromedriver.exe")
driver=webdriver.Chrome(service=s)
driver.get("https://www.kijiji.ca/")

# # Locating the elements on first page
region = WebDriverWait(driver, 10).until(
        EC.presence_of_element_located((By.XPATH, "//li[@data-loc-id='100009004']/a[@href='/h-ontario/9004']"))
    )
region.click()
region2=driver.find_element(By.XPATH,"//div/ul/li/a[@href='/h-windsor-area-on/1700220']")
region2.click()

go=driver.find_element(By.XPATH,"//a[@class='button-open next']")
go.click()

# # Locating the elements on second page
category = WebDriverWait(driver, 5).until(
        EC.presence_of_element_located((By.XPATH, "//button[@class='label-1952128162']"))
    )
category.click()
realestate = WebDriverWait(driver, 5).until(
        EC.presence_of_element_located((By.XPATH, "//li[@id='SearchCategorySelector-item-3']"))
    )
realestate.click()
time.sleep(2)
search=driver.find_element(By.XPATH,"//button[@class='searchSubmit-4090601312']")
search.click()
rent = WebDriverWait(driver, 10).until(
        EC.presence_of_element_located((By.XPATH, "//a[@href='/b-for-rent/windsor-area-on/c30349001l1700220']"))
    )
rent.click()
time.sleep(5)

#Scraping the required datas and storing them into an empty list
results=[]
addresses=[]
url_datas=[]
titles=driver.find_elements(By.XPATH,"//div[@class='title']//a")
address=driver.find_elements(By.XPATH,"//span[@class='intersection'][2]")
prices=driver.find_elements(By.XPATH,"//div[@class='price']")
links=driver.find_elements(By.XPATH,"//div[@class='title']/a[@href]")
try:
    for i in range(len(titles)):
        data={'Titles': titles[i].text,
              'Address': address[i].text,
              'Prices': prices[i].text,
              'Links': links[i].get_attribute("href")}
        addresses.append(data['Address'])
        results.append(data)
    df=pd.DataFrame(results)    
except NoSuchElementException:
    results.append(None)

#Storing the list of addresses into a txt file
with open('addresses.txt','w+') as myfile:
    for item in addresses:
        myfile.write(item.replace('Marys',"Mary's") + ', Windsor' + ', Ontario' + '\n')
myfile.close()    

# Use Geopy to fetch geocode data
coordinates=[]
geolocator = Nominatim(user_agent="AKG")
with open("addresses.txt",'r') as fp:
    for line in fp:
        location = geolocator.geocode(line)
        if location is not None:
            data3={'Latitude': location.latitude,'Longitude': location.longitude}
            coordinates.append(data3)
        else:
              print(f"Could not find Location for {line}")
fp.close()
coordinates_df=pd.DataFrame(coordinates)

#Combining two dataframes
final_df=pd.concat([df,coordinates_df],axis = 1)
final_df
#After obtaining the coordinates, we can plot them on map using Folium

windsor_location=[42.314197, -83.037055]
wmap = folium.Map(windsor_location,zoom_start = 11, width=1500, height=700,control_scale=True)
locations=coordinates_df
locations_list=locations.values.tolist()

mCluster = MarkerCluster(name="Cluster Name").add_to(wmap)

#href='<a href="URL" target="_blank">mytext</a>'
university=folium.Marker(location=[42.304591479548215,-83.06623226439827],popup='University Of Windsor',icon=folium.Icon(color="red")).add_to(mCluster)

for point in range(len(locations_list)):
    folium.Marker(locations_list[point], popup=(f"<a href='{final_df['Links'][point+1]}'' target='_blank'>{final_df['Titles'][point+1]}</a>")).add_to(mCluster)  
wmap.save("index.html")   
wmap
