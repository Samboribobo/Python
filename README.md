# Python
import pandas as pd
import numpy as np
from scipy.stats import ttest_ind

# Assignment 4 - Hypothesis Testing
This assignment requires more individual learning than previous assignments - you are encouraged to check out the [pandas documentation](http://pandas.pydata.org/pandas-docs/stable/) to find functions or methods you might not have used yet, or ask questions on [Stack Overflow](http://stackoverflow.com/) and tag them as pandas and python related. And of course, the discussion forums are open for interaction with your peers and the course staff.

Definitions:
* A _quarter_ is a specific three month period, Q1 is January through March, Q2 is April through June, Q3 is July through September, Q4 is October through December.
* A _recession_ is defined as starting with two consecutive quarters of GDP decline, and ending with two consecutive quarters of GDP growth.
* A _recession bottom_ is the quarter within a recession which had the lowest GDP.
* A _university town_ is a city which has a high percentage of university students compared to the total population of the city.

**Hypothesis**: University towns have their mean housing prices less effected by recessions. Run a t-test to compare the ratio of the mean price of houses in university towns the quarter before the recession starts compared to the recession bottom. (`price_ratio=quarter_before_recession/recession_bottom`)

The following data files are available for this assignment:
* From the [Zillow research data site](http://www.zillow.com/research/data/) there is housing data for the United States. In particular the datafile for [all homes at a city level](http://files.zillowstatic.com/research/public/City/City_Zhvi_AllHomes.csv), ```City_Zhvi_AllHomes.csv```, has median home sale prices at a fine grained level.
* From the Wikipedia page on college towns is a list of [university towns in the United States](https://en.wikipedia.org/wiki/List_of_college_towns#College_towns_in_the_United_States) which has been copy and pasted into the file ```university_towns.txt```.
* From Bureau of Economic Analysis, US Department of Commerce, the [GDP over time](http://www.bea.gov/national/index.htm#gdp) of the United States in current dollars (use the chained value in 2009 dollars), in quarterly intervals, in the file ```gdplev.xls```. For this assignment, only look at GDP data from the first quarter of 2000 onward.

Each function in this assignment below is worth 10%, with the exception of ```run_ttest()```, which is worth 50%.

# Use this dictionary to map state names to two letter acronyms
states = {'OH': 'Ohio', 'KY': 'Kentucky', 'AS': 'American Samoa', 'NV': 'Nevada', 'WY': 'Wyoming', 'NA': 'National', 'AL': 'Alabama', 'MD': 'Maryland', 'AK': 'Alaska', 'UT': 'Utah', 'OR': 'Oregon', 'MT': 'Montana', 'IL': 'Illinois', 'TN': 'Tennessee', 'DC': 'District of Columbia', 'VT': 'Vermont', 'ID': 'Idaho', 'AR': 'Arkansas', 'ME': 'Maine', 'WA': 'Washington', 'HI': 'Hawaii', 'WI': 'Wisconsin', 'MI': 'Michigan', 'IN': 'Indiana', 'NJ': 'New Jersey', 'AZ': 'Arizona', 'GU': 'Guam', 'MS': 'Mississippi', 'PR': 'Puerto Rico', 'NC': 'North Carolina', 'TX': 'Texas', 'SD': 'South Dakota', 'MP': 'Northern Mariana Islands', 'IA': 'Iowa', 'MO': 'Missouri', 'CT': 'Connecticut', 'WV': 'West Virginia', 'SC': 'South Carolina', 'LA': 'Louisiana', 'KS': 'Kansas', 'NY': 'New York', 'NE': 'Nebraska', 'OK': 'Oklahoma', 'FL': 'Florida', 'CA': 'California', 'CO': 'Colorado', 'PA': 'Pennsylvania', 'DE': 'Delaware', 'NM': 'New Mexico', 'RI': 'Rhode Island', 'MN': 'Minnesota', 'VI': 'Virgin Islands', 'NH': 'New Hampshire', 'MA': 'Massachusetts', 'GA': 'Georgia', 'ND': 'North Dakota', 'VA': 'Virginia'}

def get_list_of_university_towns():
    '''Returns a DataFrame of towns and the states they are in from the 
    university_towns.txt list. The format of the DataFrame should be:
    DataFrame( [ ["Michigan", "Ann Arbor"], ["Michigan", "Yipsilanti"] ], 
    columns=["State", "RegionName"]  )
    
    The following cleaning needs to be done:

    1. For "State", removing characters from "[" to the end.
    2. For "RegionName", when applicable, removing every character from " (" to the end.
    3. Depending on how you read the data, you may need to remove newline character '\n'. '''
    
    #Importung University towns dataset as a CSV
    ut = (pd.read_csv('university_towns.txt',
                    sep='/n',
                    engine ='python',
                    header = None,)
         .rename(columns= {0:'State'})) #Renaming the first column in the dataframe to 'State'
    
    #Copying over information from the 'State' column to a new column 'RegionName'
    #except for rows that contain '[edit]'
    ut.loc[~ut['State'].str.contains("[edit]", regex=False), 
          'RegionName'] = ut['State']
    
    #Logical expression to convert row entries to np.nan in instances where the 'State' and 
    #'RegionName' columns have the same vakues 
    ut.loc[ut['State']==ut['RegionName'],
          'State'] = np.nan
    
    #Getting rid of the square brackets and anything within them in the 'State' column while filling the missing
    #values with the value in the row prior
    ut['State'] = (ut['State'].str.replace(r'\[.*','')
                  .fillna(method = 'ffill'))
    
    #Getting rid of values within parantheses and square brakckets
    ut['RegionName'] = ut['RegionName'].str.replace(r'\W*\(.*','')
    #alternatively:     ######### ut['RegionName'] = ut['RegionName'].str.replace(r'\[.*\]','')
                        ######### ut['RegionName'] = ut['RegionName'].str.replace(r'\(.*\)','')
        
    #dropping na values from the 'RegionName' column 
    ut = ut.dropna()
    
    return ut

get_list_of_university_towns()

'''Returns a DataFrame of GDP over time from the gdplev.xls file.'''
def get_gdp_time_series():       
    #importing GDP overtime in the United States dataset
    gdp = pd.read_excel('gdplev.xls', header = None, skiprows = 5, usecols=[4,6])
    
    #Renaming headers
    new_header = gdp.iloc[0] #grab the first row for the header
    gdp = gdp[1:] #take the data less the header row
    gdp.columns = new_header #set the header row as the df header
    
    #Renaming first column 
    ## alt : gdp.columns.values[0] = 'Period'
    gdp = gdp.rename(columns = {np.nan : 'Period', 
                                'GDP in billions of chained 2009 dollars': 'GDP'})
    
    #Taking a subset of the dataframe from 2000q1 only
    gdp = gdp[gdp['Period']>'2000'].copy()   
    
    return gdp
get_gdp_time_series()

def get_recession_start():
    '''Returns the year and quarter of the recession start time as a 
    string value in a format such as 2005q3'''
    
    r = get_gdp_time_series()
    
    rec_start = r[(r['GDP'] > r['GDP'].shift(-1))&
                  (r['GDP'].shift(-1) > r['GDP'].shift(-2))].copy()
    
    return rec_start.iloc[0,0]
get_recession_start()

def get_recession_end():
    '''Returns the year and quarter of the recession end time as a 
    string value in a format such as 2005q3'''
    
    r = get_gdp_time_series()
    rec_start = get_recession_start()
    r = r[r['Period'] >= rec_start].copy()

    rec_end = r[(r['GDP'] > r['GDP'].shift(1)) &
                (r['GDP'].shift(1) > r['GDP'].shift(2))].copy()
    
    return rec_end.iloc[0,0]
get_recession_end()

def get_recession_bottom():
    '''Returns the year and quarter of the recession bottom time as a 
    string value in a format such as 2005q3'''
    #using the function that imports the gdp time series dataset
    r = get_gdp_time_series()
    
    #defining variavles for recession start and recession ends
    rec_start = get_recession_start()
    rec_end = get_recession_end()
    
    #Filtering the gdp time series data to get the period of recession 
    r = r[(r['Period'] >= rec_start) &
        (r['Period'] <= rec_end)].copy()
        
    return r.loc[r['GDP'].idxmin()]['Period']

##alt: 
    ##rec_bottom = r[r['GDP']==r['GDP'].min()].copy()
    ##return rec_bottom
get_recession_bottom()

def convert_housing_data_to_quarters():
    '''Converts the housing data to quarters and returns it as mean 
    values in a dataframe. This dataframe should be a dataframe with
    columns for 2000q1 through 2016q3, and should have a multi-index
    in the shape of ["State","RegionName"].
    
    Note: Quarters are defined in the assignment description, they are
    not arbitrary three month periods.
    
    The resulting dataframe should have 67 columns, and 10,730 rows.
    '''
    #importing housing dataset 
    hd = (pd.read_csv('City_Zhvi_AllHomes.csv'))
    #mapping the state column to the state dictionary in the code above
    hd['State'] = hd['State'].map(states)
    #setting data index as indicated in the prompt
    hd = hd.set_index(['State', 'RegionName'])
    
    #Assigning a variable for columns to drop from the dataset
    hd_drop = hd[hd.columns[4:49]]
    hd = hd.drop(hd_drop,axis=1).copy()
    
    hd_final = hd[hd.columns[4:204]]
    hd_final = hd_final.groupby(pd.PeriodIndex(hd_final.columns, freq='Q'), axis=1).mean()

    return hd_final
convert_housing_data_to_quarters()

def run_ttest():
    '''First creates new data showing the decline or growth of housing prices
    between the recession start and the recession bottom. Then runs a ttest
    comparing the university town values to the non-university towns values, 
    return whether the alternative hypothesis (that the two groups are the same)
    is true or not as well as the p-value of the confidence. 
    
    Return the tuple (different, p, better) where different=True if the t-test is
    True at a p<0.01 (we reject the null hypothesis), or different=False if 
    otherwise (we cannot reject the null hypothesis). The variable p should
    be equal to the exact p value returned from scipy.stats.ttest_ind(). The
    value for better should be either "university town" or "non-university town"
    depending on which has a lower mean price ratio (which is equivilent to a
    reduced market loss).'''
    
    return "ANSWER"
