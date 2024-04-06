# Goals
![image](https://github.com/vamsikunapar/Dynamic-Pricing-Airbnb/assets/140461124/50664ca6-034c-4542-bfbc-72eae5a12c59)


# Assumptions
**Overall Model**
- Our Airbnb listings compete with other Airbnbs, not hotels, as we assume customers are only comparing Airbnb options.
- Prices vary by month, weekday, weekend, and events due to demand changes. But within these groups, prices are constant, making our data time-insensitive.
- One Airbnb reservation is allowed per rental period (based on Airbnb’s own policy).
- We only consider sellers who've owned the same property for a while, assuming they can easily obtain necessary listing data.

**Variables**
- The parking capacity is reflective of the capacity of the property i.e. the need for excess parking by a renting party was not considered. 
- If greater than two guests belong to a renting party, costs increase by an additional $X per person + $X(G-2). This generalizes the increase in cost based on additional members of the rental party.
- Monthly upkeep includes mortgage payment (or rent), utilities, cleaning fee, HOA fees, etc. and will be used as a single value to approximate the monthly cost to operate the rental.

# Data Sources
Airbnb Listing information
http://insideairbnb.com/get-the-data.html

An additional dataset that is from the San Francisco public safety database for all reported police department incidents to model safety scores
https://data.sfgov.org/Public-Safety/Police-Department-Incidents/tmnf-yvry

# Data Model
## Occupancy Rate 
As found in the Overview of the Airbnb Community in San Francisco published by Airbnb, the average length of stay per guest is 4.2 nights. We assumed each listing has 4.2 days as an average lengths of stay per booking. Since we were not able to find a clear number for the ratio  of guests making a booking who leave a review for Airbnb, we assumed the review rate to be equal to 0.5, which will be used as a constant throughout the estimation. To prevent artificially high results, we also assumed the maximum occupancy rate cannot exceed 0.95, meaning even the busiest of listings will have several nights a month in which they go unrented. With these assumptions and constants, we generated the formulation of estimated occupancy rate shown below:

![alt text](https://user-images.githubusercontent.com/30711638/48401720-c3bc2c80-e6f7-11e8-84e7-82cd1dbe25dd.png)


## Demand Function
Initially, we planned to have two separate models for predicting the listing price and the demand. One such model used k-Means to cluster all Airbnb listings within 3 miles radius to find groups of nearby similar properties, and took the average occupancy rate to estimate the expected monthly demand within each cluster. We then had another independent k Nearest Neighbor regression model to forecast daily listing price based on listing attributes that best represent the property values (amenities, safety, etc). Finally, we estimated the expected monthly profit by taking the daily average listing price of each month and multiplied by the predicted monthly demand ( the predicted number of days that a property is booked in a month).  
However, after many attempts to improve this model’s accuracy, we found that it was unreasonable to predict demand and listing price separately, primarily because these two quantities are heavily correlated with each other. For example, the demand will likely decrease if the listing price increases, since customers can be expected to make smart decisions and switch their booking to a cheaper Airbnb with similar characteristics. Therefore, we decided to model demand first, and then incorporate it into our price function. This meant that we needed to construct a demand function that captures competitor factors, listing characteristics, and time fluctuations.

## Competition Analysis
We model the competitor factor by using the same approach that we initially planned to apply for our monthly demand estimation. Customers tends to choose their Airbnb with a specific location in mind, so all listings that are located close together (within 3-mile radius) will be more likely to compete with each other. Going beyond this, the characteristics and quality of the listing should have an almost equal, and occasionally greater impact on determining occupancy rates relative to those properties competing with one another. From these guides we elected to use k-Means to cluster similar properties within a 3-mile radius distance (figure 2&3). 

## Formulate Demand Function of “Competition” Group
For each cluster, we have a set of data with the X-variable being the listing price and the Y-variable being the demand represented by the occupancy rate. From this we will fit either a  linear or polynomial regression model onto this dataset to find the best-fitted function of demand  
$D_{xij} = \alpha* x_{ij}^2 + \beta*x_{ij} + c$.  
We then put this demand function into the optimization model. The objective is to maximize profit in one year, so the formula to calculate profit for each day is the listing price on that day multiplied by the demand function on that day, represented as $D_{xij} * x_{xij}$. It is worth noting that at this point in the analysis, demand in our dataset is the occupancy rate, which represents the probability a property is booked on a given day. We will mention the details of how to calculate the optimal yearly profit in the Decision Model later. 
	After several trials, we determined that the relationship between demand and price is not always inversely correlated together and straightforward linear; outlier-type cases in particular don’t fit into our model, meaning that extremely expensive listings seem to have different rules of demand governing them, which is reasonable but not accounted for in our limited model. 

 ## Prediction of Intrinsic Property Values
We first set out to incorporate the property-specific characteristics into the model to customize the pricing model for each listing; as mentioned previously if we include too many variables into an optimization, we are less likely to have a valid solution. Instead we used our regression model to determine a “baseline” price, which we define as a price mark that indicates the intrinsic value of an Airbnb property without considering time series fluctuation (seasonalities, weekend vs weekday). Then in the optimization model we display the daily price $$x_{ij}$$  in terms of the defined baseline price with the constraint  
$$x_{ij} = \gamma_{ij} * x_0$$  
The optimization model will find the optimal coefficient $\gamma_{ij}$, and consequently the optimal daily listing price xij. 
	Additionally we used the k Nearest Neighbor (kNN) regression to determine the most accurate fitted baseline price for each listing, based on attributes that we decided will determine the value of a listing such as location, amenities, review scores, safety, etc. The reason we selected the kNN algorithm is because it determines the response variable Y based on the values of X-variables from k neighbors, which in our case is the values of nearby Airbnb competitors. This is a good model fit for our data because in real life, it is usually the case that real estate, hotel, and Airbnb values are heavily influenced by their surrounding competitors as a function of their location.

# Optimization Model
### Time series factors
In attempting to find the daily optimal price, we determined that our optimization model will have up to 365 price variables x1,..., x365. We originally simplified the model by assuming that price is relatively consistent throughout one month. However when further analyzing past data, we observed that Airbnb’s demand is generally higher during weekend than weekday, so we further delineated our model into two separate optimization models for both weekends and weekdays. From this we arrived at our model’s assumptions that listing price should be constant for all weekends within a month and for all weekdays within a month. 

### Decision Variables:
- $\alpha_{ij}$: binary variable which is 1 if customers decide to book Airbnb property on day j of month i, equal to 0 otherwise
- $x_{ij}$: price of Airbnb property on day j of month i
Parameters:
- $C_V$ is variable cost that includes cleaning fee, utilities, guest-included fee, deterioration fee on amenities. For simplicity of our model, we will let users independently input $C_V$ to the application, allowing us to treat $C_V$ as a constant. Moreover, we assume $C_V$ will be incurred only on the days where bookings happened.  
- $I_O$ is initial investment the property owner supplies, which potentially includes the real estate or reimbursement costs, amenity purchases, maintenance cost or any other upfront cost. For simplicity of our model, we will let users input the estimate initial investment on their property, allowing us to treat this as a fixed cost throughout our model.  
- $D_{ij}$: demand of day j on month i, represented by the probability that the listing is actually booked on a specific day, given the listing price xij. $D_{ij}$ is the function of $x_{ij}$ and is determined by performing regression analysis on the cluster that the specific Airbnb property belongs to.
### Objective Function:
$$
Max Y = \sum_{i=1}^{12}\sum_{j=1}^{2} \alpha_{ij}*x_{ij} - \alpha_{ij}*C_{v}
$$
### Constraints:
- If $D(x_{ij})$ ≥ 0.50, then $α_{ij}$ = 1; Otherwise if $D(x_{ij})$ < 0.50, then $α_{ij}$ = 0

$$
D(x_{ij})>0.5 -M(1-\alpha_{ij})
$$

$$
D(x_{ij})\leq  0.5 +M(\alpha_{ij})
$$
- The daily Price is a function of base line price $x_{O}$

$$
x_{ij} = \beta_{ij}*x_{O}
$$

- variable Cost = Cleaning fee + Security Deposit +Guest -included fee

$$
C_{V} = C_{clean} +C_{security} + C_{guest}
$$

- Initial investment must be broken in 12 months

$$
Y \geq I_{O}
$$

### Output

We tested two example inputs in our optimization model. The first example is the “Boutique Hotel” property in the Bayview neighborhood. The k-Means clustering algorithms classify this listing into a cluster whose average prices are relatively high compared to other clusters of San Francisco Airbnbs. After inspecting the detailed attribute values of these listings within this cluster, we can observe that most of them have safety score above 7, and amenity scores that are majority ‘luxury’. We can therefore label this cluster as “high-end” listing group. From this cluster, we perform a kNN regression to find the demand function as outlined in our model above. The regression model predicts the intrinsic value of this property to be $132, which means independent from all the time factors, this is the ideal price we should list for. The optimization model gives us 24 values of decision variables, which are the recommended prices of both weekends and weekdays in 12 months. For example, t4,1 is the recommended price of all weekends in April. We take the sum of recommended price on the day that optimization model predicts bookings will happens and subtract the expected cost to generate our expected yearly profit of $25,378. 

Similarly, we choose one listing in the Chinatown with “Bed and Breakfast” motif. This listing belongs to a seemingly “medium-category” Airbnb as evidenced by it’s lower safety score and lower intrinsic value score, meaning that it’s amenities offered are less appealing than those of the Bayview listing. Our demand rate is also lower than the Bayview’s; it seems that despite a potentially lower price, the Chinatown listing is in a location that was less popular among prior Airbnb users. The remaining calculation process is similar to the previous example, and just as one would expect from the above inputs, the yearly net profit is lower than the Bayview example, at $19,764.

### Potential Issuses
- Issue 1: When users post a new listing, they have a field to fill in information for all the attributes related to the listings. While in a real life setting only a small subset of these attributes are “required”, because we include the optional attributes in our regression model, the model won’t work properly for that listing if the users don’t fill in all possible attributes. 
- Issue 2: We assume that within a given month, the prices of the weekend and weekday, except special-event days, are the same, which can definitely cause problems in this model moving forward. The demand is a key factor to determine the dynamic changes of the listing price, and the demand of Airbnb in any given area could easily fluctuate throughout weekdays or within the same month. 
- 
### Methods to monitor the potential occurrence of the issues 
- Issue 1: Whenever a new listing is submitted to our system, we would implement a script to check whether all the attributes are filled. If any of these attributes are left empty, it would send a notification to the user with the ids of the listings with empty attributes.
- Issue 2: A possible method for monitoring this that we discussed was to monitor the number of flights arrived to San Francisco airport, as it is generally agreed that air traffic is a decent indicator for the travel flow of San Francisco. If during the week, the number of flights arrived fluctuates greatly (we can have a condition to check whether the moving average of flights exceed a certain benchmark to represent whether the fluctuation is significant or not), then the system will send us a notification.

### Methods for fixing/ resolving the issues 
- Issue 1: All the missing attributes are, by definition, optional information, so it’s not crucially important to have those fields. We can auto-fill these attributes with the most common values (for categorical variables) or with the median values (for numerical variables) scraped from the Airbnb dataset. This means that if the reviews are blank, then we can autofill the reviews of those listings with the median values of all other reviews that have been filled. 
- Issue 2: If we get the notification to indicate that the demand, represented by travel, is undergoing significant fluctuation, the engineering team then has to more thoroughly inspect the demand trend to see the extent of the fluctuations, or any possible reasons for the fluctuations. If the frequency of the notification is high within a day or a week, we could change our model from estimation based on clustering to the previously discussed time-series regression model to predict the daily demand of each Airbnb based on their characteristics. 
