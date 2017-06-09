# GitEducation

Content

[Basket](#basket) [see tn](/techNotes.md#basket) 

[Some Text][]

[Link to Header](#the-header)

Checkout

Payment

Customer


### Bulk products

#### Bulk Item Specific Exceptions
If amount of money spent for bulk products in the order doesn't achieve the minimum order value(**/kf/commerce/pricing/charges/tools/ComplexDeliveryChargeConfiguration.minimumOrderValue**) **BulkDeliveryException** will be present in the response.


### Customer

#### Patch customer

**Email Sending functionality**

Customer PATCH operation sends an email notification to the customer. The notification is set to both new and old email addresses. **MarketingInfo** token is regenerated as well. There are some restrictions for the operation.  
* new email must be valid
* new email mustn't be present in DB
* user login will be changed according to the new email e.g. **BQ_the@new.email**, so next time the customer should use new credentials


#### Guest customer

If a customer trying to authentificate using "**GUEST**" password(compared to value of **com/kingfisher/group/api/service/util/ProfileHelper.guestPassword**), 
then the specified login should be a guest login. It means that 
* **onlineStatus** property of corresponding profile can't be **Initialized** or **Locked**
* **guestCheckoutUser** property of corresponding profile must be **true**
In such case authorization token is generated.
Otherwise, the corresponding exception is thrown.

### Basket

#### Click&Collect products

In order to be available for **Click&Collect** SKU must meet following requirements:
* SKU property **shippingClasses**(property of **sku** descriptor in **atg/commerce/catalog/ProductCatalog** repository) contains list of ids that correspond to **shippingMethod** descriptor of **kf/repository/KFRepository** repository and there are Click&Collect methods among the list(see **shippingMethodType** property of **shippingMethod** descriptor). **Brand** and **Channel** properties are taken into account as well
* the user store(**store** property of **user** descriptor of **atg/userprofiling/ProfileAdapterRepository** points to **id** of **store** descriptor in **atg/store/stores/StoreRepository**) is available for **Click&Collect**(see **storeProfile** of **store** descriptor; **Store Pickable** corresponds to **CCSameDay**, **Central Receivable** corresponds to **CCNextDay**)
* in order to be available for **CCSameDay**, **storeInventory**(**storeInventory** descriptor of **atg/commerce/inventory/InventoryRepository**) must have an item that indicates stock level for the sku in the store

### Checkout


#### Click&Collect products

#### Preconditions


#### Collection date

API call:
>**POST {{apiEnv}}/groupapi/v1/checkout/BQUK/{{basketID}}/fulfilmentGroups**

returns **CollectionDate**

If store is a **restricted** one, collection time is set to 5:00 pm (see **/com/kingfisher/group/api/service/util/FulfilmentGroupHelper.specialCollectionHours** prooerty) instead of 12:00 pm (see **/com/kingfisher/group/api/service/util/FulfilmentGroupHelper.defaultCollectionHours** prooerty).

**Restricted** stores are ones that present in **/kf/commerce/Configuration.selectedStoresId** or **/kf/commerce/Configuration.niIomCcStores** properties.

**CCSameDay**
* collection time is the order placement time +1 hr (rounded up to nearest 10 minutes)
* if ordered after cutoff (19:00 weekdays – 16:00 Sat and Sun) then CollectionDate is shop's opening time +1 hr tomorrow
* if product has both **CCSameDay** and **CCNextDay** then priority is given to **CCSameDay** shipping method

If **TrainingReadyStore**(property of **store** item in **/atg/registry/ContentRepositories/StoreRepository**) is true, it means that the CCSameDay training for the store hasn't finished and following rules are applied:
* for regular stores - one more day is required, so CollectionDate is 12:00 pm +1 day
* for selected(present in **/kf/commerce/Configuration.selectedStoresId**) non-**IOM** stores - one more day is required, so CollectionDate is 5:00 pm +1 day
* for **IOM**(**NI** and the three stores:**Inverness**, **Peterhead** and **Elgin**, present in **/kf/commerce/Configuration.niIomCcStores**) 5:00 pm +2 days
* if delivery day is a holiday, CollectionDate is +1 day

**TrainingReadyStore** is applicable for **CCSameDay** only.

**Restricted** store functionality for **CCSameDay** works only in case of **TrainingReadyStore** is **true**.

The +2 days functionality can be switched off(+1 day will used instead) by setting **/kf/common/featuretoggles/DefaultFeatureToggles.showSamedayNIAndIOMMessage=false**.

**CCNextDay**
* collection time is next day 12:00 pm
* if delivery day is a holiday then +1 day

Note: Looks strange that this functionality allows obtaining **CCNextDay** product even earlier then **CCSameDay** one because TrainingReadyStore doesn't affect **CCNextDay**.

### Some Text ###


## The Header



