Content

### Bulk products

#### Bulk Item Specific Exceptions
ATG component **/kf/commerce/pricing/charges/tools/ComplexDeliveryChargeConfiguration** is responsible for delivery charge configuration. **/com/kingfisher/group/api/service/util/BasketApiModelHelper** uses it in **mapBulkDeliveryException** method. Order bulk total is calculated via **/com/kingfisher/group/api/service/util/BasketApiHelper.fetchBulkOrderTotal** method.
If order bulk total doesn't achieve **ComplexDeliveryChargeConfiguration.minimumOrderValue** value, **com.kingfisher.group.api.model.BulkDeliveryException** is created and set into **com.kingfisher.group.api.model.BasketAttributes**.


### Customer

#### Patch customer

#### Email Sending functionality

There are some API calls which can send email notification to the customer. For example, there is email change notification done in scope of Customer PATCH operation when the email is updated. The example of email sending functionality might be found in **com.kingfisher.group.api.messaging.email.service.CustomerEmailUpdateServiceImpl.sendEmailNotification** method.

All base classes which are related to the core email sending functionality might be found inside __com.kingfisher.group.api.messaging.email.*__  package. The central role of sending emails is played by **com.kingfisher.group.api.messaging.email.GroupAPIEmailSenderHelper**. This class creates email template according to the passed parameters to the **sendEmail** method. There is following behavior of email template resolving which depends on the siteId and **EMAIL_TEMPLATE_NAME** method parameters. For example, how **sendEmail** function should be used, is shown below:
```java
final Map<String, String> emailParamMap = new HashMap<String, String>();
emailParamMap.put(FIRST_PARAM_NAME, firstParamValue);
emailParamMap.put(SECOND_PARAM_NAME, secondParamValue); 

File[] attachments = new File[1];
pAttachments[0] = someAttachedFile; 

boolean isPersistEmail = true; 

final GroupAPIEmailSenderHelper.UserInfo userInfo = new GroupAPIEmailSenderHelper.UserInfo(customerId, customerEmailToSend, brand, channel);
final GroupAPIEmailSenderHelper.EmailParam emailParam = new GroupAPIEmailSenderHelper.EmailParam(EmailTemplatesConstants.EMAIL_TEMPLATE_NAME, emailParamMap, attachments, isPersistEmail); // attachment might be null
this.emailSenderHelper.sendEmail(userInfo, emailParam);
```	
An email template should be an ATG component which is based on the **kf.sitebuilder.email.KFSiteBuilderTemplateEmailInfo**. This template assume that the SiteBuilder asset should be used as an email template (Example of email template configuration might be found on the following ATG component **/com/kingfisher/group/api/messaging/email/template/CustomerEmailChangeTemplate**).
There is next behavior of email template resolving depend on the siteId and email template name:

1. GroupAPIEmailTemplateResolver. This ATG component connects specific site id with its email template holder, for example for diyStore it is **DIYStoreEmailTemplatesHolder** component.
2. Site specific EmailTemplateHolder should be defined based on the **com.kingfisher.group.api.messaging.email.EmailTemplatesHolder**. This component connect **EmailTemplatesConstants.EMAIL_TEMPLATE_NAME** with corresponding ATG email template component for specific siteId.
3. There are also components (like **ChangeEmailTemplateSender**) which configure the EmailTemplate message subject in another config file (for example on the **KFEmailSubjects** properties file). To re-use this configuration there is **emailSubjectKeysMap** inside **GroupAPIEmailSenderHelper**. Therefore, if the emailSubjectKeysMap contains **EMAIL_TEMPLATE_NAME** then the email subject message will be taken from configured component in this map.


#### Guest customer

The main logic is in **com/kingfisher/group/api/service/util/ProfileHelper.retrieveProfileAuthInfo** method
```java
if(this.guestPassword.equals(password)) {
	userProfile = this.identityManager.getUserItem(login);

	if(Objects.isNotNull(userProfile) && isGuest(userProfile)) {
		profileAuthInfoDTO.withAuthStatus(GUEST);
	}
}
```

### Basket

#### Click&Collect products


### Checkout

#### Collection date

API call:
>**POST {{apiEnv}}/groupapi/v1/checkout/BQUK/{{basketID}}/fulfilmentGroups**

returns **CollectionDate**

Mainly the logic is implemented in **com/kingfisher/group/api/service/util/FulfilmentGroupHelper** class.
```java
public static final String SHOW_SAMEDAY_NI_AND_IOM_MSG = "showSamedayNIAndIOMMessage";
public static final int ADDITIONAL_DAYS_FOR_NI_IOM_STORES = 2;
public static final int ADDITIONAL_DAYS_FOR_RESTRICTED_STORES = 1;

private void addCollectionDateInfo(final FulfilmentGroupDTO fgDto, final KFHardgoodShippingGroup sg,
		final KFOrderImpl order) throws DetailedException {
	StoreInfo storeInfo = getStoreInfo(order);
	Calendar calendar = Calendar.getInstance(TimeZone.getTimeZone(getTimeZone()));

	if(isCCSameDayShippingGroup(sg)) {
		if(storeInfo.isTrainingReadyStore()){
			RepositoryItem storeItem = this.storeHelper.fetchStoreByExternalId(storeInfo.getExternalStoreId());
			Date collectionDate = getFakeSameDayDateForCCItem(storeItem, sg.getPromisedCollectionDateTime().getTime());
			calendar.setTime(collectionDate);
			correctTimeForCCNextDay(calendar, sg, storeInfo);
		}
		else{
			calendar.setTime(sg.getPromisedCollectionDateTime().getTime());
		}
	}

	else {
		calendar.setTime(sg.getActualDeliveryDate());
		correctTimeForCCNextDay(calendar, sg, storeInfo);
	}

	fgDto.setCollectionDate(calendar.getTime());
}

private Date getFakeSameDayDateForCCItem(final RepositoryItem storeItem, final Date retrieveDate) {
	final Calendar calendar = Calendar.getInstance();
	calendar.setTime(retrieveDate);

	String storeId = (String)storeItem.getPropertyValue(KFPropertyNames.STORE_ID);

	if(this.featureToggleService.isFeatureToggled(SHOW_SAMEDAY_NI_AND_IOM_MSG) && Objects.isNotEmpty(storeId) &&
			this.configuration.getNiIomCcStores().contains(storeId)) {
		calendar.add(Calendar.DAY_OF_YEAR, ADDITIONAL_DAYS_FOR_NI_IOM_STORES);
	} else {
		calendar.add(Calendar.DAY_OF_YEAR, ADDITIONAL_DAYS_FOR_RESTRICTED_STORES);
	}

	final Date fakeCCSameDayDeliveryDate = this.messageTools
				.retrieveDate(KFConstants.CHECKOUT_MESSAGE, calendar.getTime(), storeItem);

	return fakeCCSameDayDeliveryDate;
}

private Calendar correctTimeForCCNextDay(final Calendar calendar, final KFHardgoodShippingGroup sg, final StoreInfo storeInfo){
	int hourValue = getDefaultCollectionHours();

	if(Objects.isNotNull(storeInfo) && Objects.isNotNull(storeInfo.getId())) {
		if(isRestrictedStore(storeInfo.getId())) {
			hourValue = getSpecialCollectionHours();
		}
	}
	calendar.set(Calendar.HOUR_OF_DAY, hourValue);
	calendar.set(Calendar.MINUTE, 0);
	calendar.set(Calendar.SECOND, 0);
	calendar.set(Calendar.MILLISECOND, 0);

	return calendar;
}
```

Notes:

**Restricted** stores are ones that present in **/kf/commerce/Configuration.selectedStoresId** or **/kf/commerce/Configuration.niIomCcStores** properties.

In order be **CCSameDay** or **CCNextDay** sku should have appropriate **shippingClasses**. If **shippingClasses**are changed, 
**kf/product/ProductFlagCache.flush** method should be executed.

TODO:

ADDITIONAL_DAYS values should have been made configurable.

### Stores

Repository: **atg/registry/ContentRepositories/StoreRepository**

#### Create a store

Example
```
<add-item item-descriptor="store" id="BQ_ELG657">
  <set-property name="regionId"><![CDATA[1300029]]></set-property>
</add-item>
```

#### Create a holiday

Example
```
<add-item item-descriptor="holiday" id="100005">
  <set-property name="holidayName"><![CDATA[HAPPY Holiday]]></set-property>
  <set-property name="holidayDate"><![CDATA[4/20/2017 00:00:00]]></set-property>
  <set-property name="regions"><![CDATA[1300029]]></set-property>
</add-item>
```
