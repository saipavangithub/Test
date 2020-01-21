public without sharing class OSIS_AssetTriggerHelper {
    public static Boolean isRecurrsive = false;
    private static Set<String> serialNumberFields = new Set<String>{'OSIS_Serial_Number__c','OSIS_Equipment_Number__c'};

   
/**********************************************************************************************
* Method Name:updateAccountOnAsset
* Purpose: This method will used to update account on asset for multi site quote.
***********************************************************************************************/
    public static void updateAccountOnAsset(List<Asset> newAssets){
        try{
            Set <Id> revisedAssetsIds = new Set<Id>();
            Map <Id,Id> quoteLineWithAccountId = new Map<Id,Id>();
           
            for (Asset currentAsset : newAssets) {
                if (String.IsNotEmpty(currentAsset.SBQQ__QuoteLine__c)) {
                    revisedAssetsIds.add(currentAsset.SBQQ__QuoteLine__c);
                }
            }
            if (revisedAssetsIds.size()  > 0) {
                for (SBQQ__QuoteLine__c quoteLine : [SELECT Id,SBQQ__Group__c,SBQQ__Group__r.SBQQ__Account__c
                                                     FROM  SBQQ__QuoteLine__c  
                                                     WHERE Id IN :revisedAssetsIds AND SBQQ__Group__c!=''
                                                     LIMIT 50000] ) {
                     if (String.isNotEmpty(quoteLine.SBQQ__Group__r.SBQQ__Account__c)) {
                         quoteLineWithAccountId.put(quoteLine.Id,quoteLine.SBQQ__Group__r.SBQQ__Account__c);    
                     }
                }
                for (Asset currentAsset : newAssets) {
                    if (String.IsNotEmpty(currentAsset.SBQQ__QuoteLine__c) &&
                        quoteLineWithAccountId.containsKey(currentAsset.SBQQ__QuoteLine__c)) {
                            currentAsset.AccountId = quoteLineWithAccountId.get(currentAsset.SBQQ__QuoteLine__c);
                        }  
                }
            }
        }
        catch(Exception exc){
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
   
/**********************************************************************************************
* Method Name:markAssetAsObsolete
* Purpose: This method will used to mark asset as replaced/decomission for amendments/decommission.
***********************************************************************************************/
    public static void markAssetAsObsolete(List<Asset> newAssets){
        try{
            Set <Id> revisedAssetsIds = new Set<Id>();
            Set <Id> assetsIds = new Set<Id>();
            for (Asset currentAsset : newAssets) {
                if (String.IsNotEmpty(currentAsset.SBQQ__RevisedAsset__c)) {
                    assetsIds.add(currentAsset.Id);
                }
            }
            if (assetsIds.size()  > 0) {
                Map<Id,Decimal>assetIdWithNewQuantity = new Map<Id,Decimal>();
                Map<Id,Decimal>revisedAssetIdWithQuantity = new Map<Id,Decimal>();
                Map<Id,Id>assetIdWithRevisedAssetId = new Map<Id,Id>();
                for (Asset currentAsset : [SELECT Id,Status,Product2.SBQQ__PricingMethod__c,Quantity,
                                           SBQQ__RevisedAsset__c,SBQQ__RevisedAsset__r.OSIS_Net_Quantity__c  
                                           FROM  Asset  
                                           WHERE Id IN : assetsIds
                                           LIMIT 50000] ) {

                    revisedAssetsIds.add(currentAsset.SBQQ__RevisedAsset__c);
                    assetIdWithNewQuantity.put(currentAsset.Id, currentAsset.Quantity);
                    assetIdWithRevisedAssetId.put(currentAsset.Id,currentAsset.SBQQ__RevisedAsset__c);
                    revisedAssetIdWithQuantity.put(currentAsset.SBQQ__RevisedAsset__c,currentAsset.SBQQ__RevisedAsset__r.OSIS_Net_Quantity__c);
                }
                if (assetIdWithRevisedAssetId.size() > 0) {
                    List<Asset>updateAssets = new List<Asset>();
                    for (Id assetId : assetIdWithRevisedAssetId.keyset()) {
                        Asset revisedAsset = new Asset();
                        Asset currentAsset = new Asset();
                        if (assetIdWithNewQuantity.containsKey(assetId)) {
                            currentAsset.Id = assetId;
                            if (assetIdWithNewQuantity.get(assetId) > 0 ) {
                                currentAsset.Status = Label.OSIS_Obsolete;
                            }
                            else{
                                currentAsset.Status = Label.OSIS_Reduced;
                            }
                            updateAssets.add(currentAsset);
                        }
                        if (assetIdWithRevisedAssetId.containsKey(assetId) &&
                            revisedAssetIdWithQuantity.containsKey(assetIdWithRevisedAssetId.get(assetId))){
                            revisedAsset.Id = assetIdWithRevisedAssetId.get(assetId);
                            if (revisedAssetIdWithQuantity.get(assetIdWithRevisedAssetId.get(assetId)) > 0 ) {
                                revisedAsset.Status = Label.OSIS_License_Purchased;
                            }
                            else if(revisedAssetIdWithQuantity.get(assetIdWithRevisedAssetId.get(assetId)) <= 0){
                                revisedAsset.Status = Label.OSIS_Decommissioned;
                            }
                            updateAssets.add(revisedAsset);
                        }
                   }
                    if (updateAssets.size() > 0) {
                        Integer scheduleTime = Integer.valueOf(Label.OSIS_ScheduleTime);
                        Datetime sysTime = System.now().addSeconds(scheduleTime);      
                        String chronExpression = '' + sysTime.second() + ' ' + sysTime.minute() + ' ' + sysTime.hour() + ' ' + sysTime.day() + ' ' + sysTime.month() + ' ? ' + sysTime.year();
                        System.schedule( 'UpdateDecomission ' + sysTime, chronExpression, new OSIS_AssetDecommissionedHelper(JSON.serialize(updateAssets)) );
                    }
                }
            }
        }
        catch(Exception exc){
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
/**********************************************************************************************
* Method Name:updateRequiredByToRevisedAsset
* Purpose: This method will update the requiredBy for bundles during amendments.
***********************************************************************************************/
    public static void updateRequiredByToRevisedAsset(List<Asset>currentAssests,Map<Id,Asset>oldAssetMap){
        Map<Id, Asset> revisedAssetMap = new Map<Id, Asset>();
        if (currentAssests != null) {
            Set<Id>parentIds = new Set<Id>();
            List<Asset>blockPricingAssets = new List<Asset>();
            if (null != oldAssetMap) {
                for (Asset currentAsset : currentAssests) {
                    if (currentAsset.SBQQ__RequiredByAsset__c != null &&
                        currentAsset.SBQQ__RequiredByAsset__c != oldAssetMap.get(currentAsset.Id).SBQQ__RequiredByAsset__c) {
                        parentIds.add(currentAsset.SBQQ__RequiredByAsset__c);
                    }
                }
            }
            else{
                for (Asset currentAsset : currentAssests) {
                    if (currentAsset.SBQQ__RequiredByAsset__c != null) {
                        parentIds.add(currentAsset.SBQQ__RequiredByAsset__c);
                    }
                }
            }
           
            if (parentIds.size() > 0) {
                for (Asset parentAsset : [SELECT Id,SBQQ__RevisedAsset__c, SBQQ__RevisedAsset__r.SerialNumber, SBQQ__RevisedAsset__r.OSIS_Equipment_Number__c FROM Asset WHERE Id IN : parentIds
                                          AND SBQQ__RevisedAsset__c != null]) {
                      revisedAssetMap.put(parentAsset.Id, parentAsset);
                }
                if (revisedAssetMap.size() > 0) {
                    for (Asset currentAsset : currentAssests) {
                        if (currentAsset.SBQQ__RequiredByAsset__c != null &&
                            revisedAssetMap.containsKey(currentAsset.SBQQ__RequiredByAsset__c)) {
                            String serialNumber = revisedAssetMap.get(currentAsset.SBQQ__RequiredByAsset__c).SBQQ__RevisedAsset__r.SerialNumber;
                            String equipmentNumber = revisedAssetMap.get(currentAsset.SBQQ__RequiredByAsset__c).SBQQ__RevisedAsset__r.OSIS_Equipment_Number__c;
                            String requiredByAsset = revisedAssetMap.get(currentAsset.SBQQ__RequiredByAsset__c).SBQQ__RevisedAsset__c;
                            currentAsset.SBQQ__RequiredByAsset__c = requiredByAsset;
                        }
                    }
                }
            }
        }
    }
   
   
   
/**********************************************************************************************
* Method Name:updateLicenseSizeOnRevisedAsset
* Purpose: This method will update the license size for bundles during amendments
***********************************************************************************************/
    public static void updateLicenseSizeOnRevisedAsset(List<Asset>currentAssests){
        try{
            if (currentAssests != null) {
                List<Asset>revisedAssets = new List<Asset>();
                for (Asset currentAsset : currentAssests) {
                    Asset revisedAsset  = new Asset();
                    revisedAsset.ParentId = revisedAsset.SBQQ__RequiredByAsset__c;
                    if (Label.OSIS_Block.equals(currentAsset.SBQQ__PricingMethod__c) && currentAsset.SBQQ__RevisedAsset__c != null) {
                        revisedAsset.OSIS_License_Size__c = currentAsset.OSIS_License_Size__c;
                        revisedAsset.Id = currentAsset.SBQQ__RevisedAsset__c;
                        revisedAssets.add(revisedAsset);
                    }
                }
                if (revisedAssets.size() > 0) {
                    update revisedAssets;
                }
            }    
        }
        catch(Exception exc) {
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
/**********************************************************************************************
* Method Name:updateParentIdByRequiredBy
* Purpose: This method will update parent id field on asset by required by field.
This is required to maintain the asset hirearchy.

***********************************************************************************************/
    public static void updateParentIdByRequiredBy(List<Asset>currentAssests,Map<Id,Asset>oldAssetMap){
        if (oldAssetMap != null) {
            for (Asset newAsset : currentAssests) {
                if (newAsset.SBQQ__RequiredByAsset__c != null &&
                    newAsset.SBQQ__RequiredByAsset__c != oldAssetMap.get(newAsset.id).SBQQ__RequiredByAsset__c) {
                        newAsset.ParentId = newAsset.SBQQ__RequiredByAsset__c;
                    }
            }
        }
    }    
   
/**********************************************************************************************
* Method Name:updateNetQuantityOnAsset
* Purpose: This method will update net quantity on root assets.
Net Qty will be used for renewals and customer community to display the qty.
***********************************************************************************************/
    public static void updateNetQuantityOnAsset(List<Asset>currentAssests,Map<Id,Asset>oldAssetMap){
        try{
            Set<Id> revisedAssetsIds = new Set<Id>();
            Set<Id> currentHAIds = new Set<Id>();
            List<Asset> haAssets = new List<Asset>();
            Map<Id,Decimal> revisedAssetIdWithQuantity = new Map<Id,Decimal>();
            Map<Id,String> revisedAssetIdWithCurentAssetCombinedKey = new Map<Id,String>();
            Map<Id,String> revisedAssetIdWithCombinedKey = new Map<Id,String>();
            List<String> productCodes = Label.OSIS_Special_Amend_Prod_Codes.split(',');
            Set<String> productCodesSet = new Set<String>();
            productCodesSet.addAll(productCodes);
            List<Asset> independentAssets = new List<Asset>();
            if (null != oldAssetMap) {
                for (Asset currentAsset : currentAssests) {
                    if (currentAsset.SBQQ__RevisedAsset__c != null && (
                        currentAsset.SBQQ__RevisedAsset__c !=
                        oldAssetMap.get(currentAsset.id).SBQQ__RevisedAsset__c  ||
                        currentAsset.Quantity != oldAssetMap.get(currentAsset.id).Quantity)) {
                            revisedAssetsIds.add(currentAsset.SBQQ__RevisedAsset__c);
                            if (String.isNotEmpty(currentAsset.SBQQ__CombineKey__c)) {
                                revisedAssetIdWithCurentAssetCombinedKey.put(currentAsset.SBQQ__RevisedAsset__c,currentAsset.SBQQ__CombineKey__c);
                            }
                    }
                    else {
                        independentAssets.add(currentAsset);
                    }
                    if (productCodesSet.contains(currentAsset.ProductCode)) {
                        currentHAIds.add(currentAsset.Id);        
                    }
                }    
            }
            else{
                for (Asset currentAsset : currentAssests) {
                    if (currentAsset.SBQQ__RevisedAsset__c != null) {
                        revisedAssetsIds.add(currentAsset.SBQQ__RevisedAsset__c);
                        if (String.isNotEmpty(currentAsset.SBQQ__CombineKey__c)) {
                            revisedAssetIdWithCurentAssetCombinedKey.put(currentAsset.SBQQ__RevisedAsset__c,currentAsset.SBQQ__CombineKey__c);
                        }
                    }
                    else {
                        independentAssets.add(currentAsset);
                    }
                    if (productCodesSet.contains(currentAsset.ProductCode)) {
                        currentHAIds.add(currentAsset.Id);        
                    }
                }
            }
            List<Asset> revisedAssets = new List<Asset>();
            if (revisedAssetsIds.size() > 0) {
                for (Asset childAsset : [SELECT Id,Quantity,SBQQ__RevisedAsset__c,SBQQ__RevisedAsset__r.Quantity,
                                         ProductCode,SBQQ__QuoteLine__r.SBQQ__Quote__c,
                                         SBQQ__QuoteLine__r.SBQQ__Quote__r.SBQQ__Type__c,
                                         SBQQ__RevisedAsset__r.OSIS_Net_Quantity__c,
                                         SBQQ__RevisedAsset__r.SBQQ__CombineKey__c
                                         FROM Asset
                                         WHERE SBQQ__RevisedAsset__c IN : revisedAssetsIds ]) {
                    if (childAsset.SBQQ__RevisedAsset__c != null &&
                        String.isNotEmpty(childAsset.SBQQ__RevisedAsset__r.SBQQ__CombineKey__c) ) {
                        revisedAssetIdWithCombinedKey.put(childAsset.SBQQ__RevisedAsset__c,childAsset.SBQQ__RevisedAsset__r.SBQQ__CombineKey__c);  
                    }
                    if (productCodesSet.contains(childAsset.ProductCode) ) {
                        if (null != childAsset.SBQQ__RevisedAsset__r.OSIS_Net_Quantity__c && currentHAIds.contains(childAsset.Id)) {
                            Decimal qty =  childAsset.Quantity - childAsset.SBQQ__RevisedAsset__r.OSIS_Net_Quantity__c;
                            Asset haAsset = new Asset();
                            haAsset.Id = childAsset.Id;
                            haAsset.Quantity = qty;
                            haAssets.add(haAsset);
                        }
                        revisedAssetIdWithQuantity.put(childAsset.SBQQ__RevisedAsset__c,childAsset.Quantity);
                    }
                    else{
                         if (revisedAssetIdWithQuantity.containsKey(childAsset.SBQQ__RevisedAsset__c)) {
                             Decimal qty = revisedAssetIdWithQuantity.get(childAsset.SBQQ__RevisedAsset__c);
                             qty = qty + childAsset.Quantity;
                             revisedAssetIdWithQuantity.put(childAsset.SBQQ__RevisedAsset__c,qty);
                         }
                         else{
                             Decimal qty =  childAsset.Quantity != null ? childAsset.Quantity : 0.0;
                             if(childAsset.SBQQ__RevisedAsset__c != null){
                                 qty = qty + childAsset.SBQQ__RevisedAsset__r.Quantity;    
                             }
                             revisedAssetIdWithQuantity.put(childAsset.SBQQ__RevisedAsset__c,qty);  
                         }
                    }

                }
                if (revisedAssetIdWithQuantity.size() > 0) {
                    for (Id revisedAssetId : revisedAssetIdWithQuantity.keyset()) {
                        Asset revisedAsset = new Asset();
                        revisedAsset.Id =  revisedAssetId;
                        revisedAsset.OSIS_Net_Quantity__c =  revisedAssetIdWithQuantity.get(revisedAssetId);
                        String status = Label.OSIS_License_Purchased;
                        revisedAsset.Status = status;
                        if (!revisedAssetIdWithCombinedKey.containsKey(revisedAssetId) && revisedAssetIdWithCurentAssetCombinedKey.containsKey(revisedAssetId) ) {
                            revisedAsset.SBQQ__CombineKey__c = revisedAssetIdWithCurentAssetCombinedKey.get(revisedAssetId);
                        }
                        revisedAssets.add(revisedAsset);    
                    }
                }
            }
            if (independentAssets.size() > 0) {
                for (Asset currentAsset : independentAssets) {
                    Asset independentAsset = new Asset();
                    independentAsset.id =   currentAsset.Id;
                    independentAsset.OSIS_Net_Quantity__c = currentAsset.Quantity;
                    revisedAssets.add(independentAsset);
                }  
            }
            if (revisedAssets.size() > 0) {
                update revisedAssets;
            }
            if(haAssets.size() > 0){
                updateHA_AssetsQty(JSON.serialize(haAssets));
            }
        }
        catch(Exception exc) {
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
/**********************************************************************************************
* Method Name:updateHA_AssetsQty
* Purpose: This method will update HA nodes qty.
***********************************************************************************************/
    @future
    private static void updateHA_AssetsQty(String jsonStr){
        try{
            List<Asset> haAssets = (List<Asset>) JSON.deserialize(jsonStr,List<Asset>.class);
            UPDATE haAssets;
        }
        catch(Exception exc){
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
   
/**********************************************************************************************
* Method Name:createSubscriptionAssets
* Purpose: This method will create subscription assets for newly created asset for decomisssion.
In decomission/exchange,when we are adding another product that is not a part of Bundle we
need to create subscribed asset for newly added asset as SRP will not be added for the newly
added assset.
***********************************************************************************************/
    public static void createSubscriptionAssets(List<Asset>currentAssests,Map<Id,Asset>oldAssetMap){
        try{
            Set<Id>independentOrderItemsIds = new Set<Id>();
            Set<Id>accountIds = new Set<Id>();
            Map<Id,OrderItem>orderItemIdWithOrderItem = new Map<Id,OrderItem>();
            Map<Id,Id>orderItemIdWithAssetId = new Map<Id,Id>();
            Map<Id,Id>quoteItemIdWithAssetId = new Map<Id,Id>();
            if (oldAssetMap != null) {
                for (Asset newlyCreatedAsset : currentAssests) {
                    if (newlyCreatedAsset.SBQQ__QuoteLine__c != null &&
                        newlyCreatedAsset.SBQQ__QuoteLine__c != oldAssetMap.get(newlyCreatedAsset.Id).SBQQ__QuoteLine__c) {
                            quoteItemIdWithAssetId.put(newlyCreatedAsset.SBQQ__QuoteLine__c,newlyCreatedAsset.Id);
                        }
                }
            }
            else{
                for (Asset newlyCreatedAsset : currentAssests) {
                    if (newlyCreatedAsset.SBQQ__QuoteLine__c != null) {
                        quoteItemIdWithAssetId.put(newlyCreatedAsset.SBQQ__QuoteLine__c,newlyCreatedAsset.Id);
                    }
                }
            }
            if (quoteItemIdWithAssetId.keyset().size() > 0) {
                for (OrderItem orderItemRecord : [  SELECT Id,SBQQ__QuoteLine__c,
                                                    SBQQ__QuoteLine__r.SBQQ__Quote__r.OSIS_Is_Decommission_Exchange__c,
                                                    SBQQ__QuoteLine__r.SBQQ__Quote__r.SBQQ__Account__c,SBQQ__Asset__c
                                                    FROM OrderItem WHERE SBQQ__QuoteLine__c IN : quoteItemIdWithAssetId.keyset()
                                                    AND SBQQ__QuoteLine__r.SBQQ__Quote__r.OSIS_Is_Decommission_Exchange__c = true]) {
                    if (orderItemRecord.SBQQ__QuoteLine__r.SBQQ__Quote__r.OSIS_Is_Decommission_Exchange__c) {
                        independentOrderItemsIds.add(orderItemRecord.Id);
                        if (null != orderItemRecord.SBQQ__QuoteLine__r.SBQQ__Quote__r.SBQQ__Account__c) {
                          accountIds.add(orderItemRecord.SBQQ__QuoteLine__r.SBQQ__Quote__r.SBQQ__Account__c);
                          orderItemIdWithOrderItem.put(orderItemRecord.Id,orderItemRecord);  
                        }
                        if (!orderItemIdWithAssetId.containsKey(orderItemRecord.Id) &&
                          quoteItemIdWithAssetId.containsKey(orderItemRecord.SBQQ__QuoteLine__c)) {
                              orderItemIdWithAssetId.put(orderItemRecord.Id,quoteItemIdWithAssetId.get(orderItemRecord.SBQQ__QuoteLine__c));
                        }
                    }
                }
            }
            if (independentOrderItemsIds.size() > 0) {
                Map<Id,Id>accountIdWithSubscriptionId  = new Map<Id,Id>();
                for (SBQQ__Subscription__c sub : [SELECT Id,SBQQ__Account__c,SBQQ__Product__r.ProductCode,
                                                  SBQQ__RenewedDate__c,SBQQ__StartDate__c,SBQQ__EndDate__c
                                                  FROM SBQQ__Subscription__c
                                                  WHERE SBQQ__Account__c IN : accountIds AND
                                                  SBQQ__Product__r.ProductCode = : Label.OSIS_SRP AND
                                                  SBQQ__RenewedDate__c = null  AND
                                                  SBQQ__EndDate__c >=  TODAY ORDER BY
                                                  CreatedDate DESC LIMIT 1]) {
                                                      accountIdWithSubscriptionId.put(sub.SBQQ__Account__c,sub.Id);
                                                  }
                List<SBQQ__SubscribedAsset__c>subscAssets = new List<SBQQ__SubscribedAsset__c>();
                for (Id orderItemId : independentOrderItemsIds) {
                    SBQQ__SubscribedAsset__c subAsset = new SBQQ__SubscribedAsset__c();
                   
                    if (orderItemIdWithOrderItem.containsKey(orderItemId)) {
                        if (null != orderItemIdWithAssetId.get(orderItemId) &&
                            null != orderItemIdWithOrderItem.get(orderItemId).SBQQ__QuoteLine__r.SBQQ__Quote__r.SBQQ__Account__c
                            && accountIdWithSubscriptionId.containsKey(
                                orderItemIdWithOrderItem.get(orderItemId).SBQQ__QuoteLine__r.SBQQ__Quote__r.SBQQ__Account__c)) {
                                    subAsset.SBQQ__Asset__c = orderItemIdWithAssetId.get(orderItemId);
                                    subAsset.SBQQ__Subscription__c = accountIdWithSubscriptionId.get(
                                        orderItemIdWithOrderItem.get(orderItemId).
                                        SBQQ__QuoteLine__r.SBQQ__Quote__r.SBQQ__Account__c);
                                    subAsset.SBQQ__Active__c = true;
                                    subscAssets.add(subAsset);
                                }
                    }
                }
                if (subscAssets.size() > 0) {
                    insert subscAssets;
                }
            }  
        }
        catch(Exception exc) {
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
   
   
/**********************************************************************************************
* Method Name:mapOrderLinesToAssets
* Purpose:  This method will map order line items to assets.
It will be used to map serial# and equipment# from order line to assets in case
of Try & Buy order.
***********************************************************************************************/
    public static void  mapOrderLinesToAssets(List<Asset>currentAssests){
        try{
            Set<Id>orderItemsIds = new Set<Id>();
            Map<String,Asset>orderItemIdWithAsset = new Map<String,Asset>();
            for (Asset newlyCreatedAsset : currentAssests) {
                if (newlyCreatedAsset.SBQQ__OrderProduct__c != null) {
                    orderItemsIds.add(newlyCreatedAsset.SBQQ__OrderProduct__c);
                    orderItemIdWithAsset.put(newlyCreatedAsset.SBQQ__OrderProduct__c,newlyCreatedAsset);
                }
            }
            Map<String,String>defaultValues = new Map<String,String>();
            Map<String,String> parameters = new Map<String,String>();
            if (orderItemsIds.size() > 0) {
                for (OSIS_Quote_Order_Mapping__mdt assetOrderMap : [SELECT Id,OSIS_Default_Value__c,OSIS_Source_Field__c,
                                                                    OSIS_Source_Object__c,OSIS_Target_Object__c,
                                                                    OSIS_Target_Field__c FROM OSIS_Quote_Order_Mapping__mdt
                                                                    WHERE OSIS_Source_Object__c = 'OrderItem' AND
                                                                    OSIS_Target_Object__c = 'Asset'] ) {
                    if (String.isNotEmpty(assetOrderMap.OSIS_Default_Value__c)) {
                        defaultValues.put(assetOrderMap.OSIS_Target_Field__c,assetOrderMap.OSIS_Default_Value__c);  
                    }
                    if (String.isNotEmpty(assetOrderMap.OSIS_Source_Field__c)) {
                        parameters.put(assetOrderMap.OSIS_Source_Field__c,assetOrderMap.OSIS_Target_Field__c);          
                    }  
                }
                List<String>fieldSet = new List<String>();
                if ((parameters != null  && parameters.size() > 0)) {
                    fieldSet.addAll(parameters.keyset());
                }
                Set<String>orderLineItemIds = orderItemIdWithAsset.keyset();
                String queryStr = 'SELECT id,OrderId,Order.OSIS_Transaction_Type__c,';
                queryStr = queryStr + String.join(fieldSet,',')+ ' FROM OrderItem WHERE';
                queryStr = queryStr + ' Id IN: orderLineItemIds' ;
                Map<Id,OrderItem> IdWithOrderLines = new Map<Id,OrderItem>();
                for (OrderItem orderLine : Database.query(queryStr)) {
                    IdWithOrderLines.put(orderLine.Id,orderLine);
                }
                for (String orderLineId : IdWithOrderLines.keyset()) {
                    // Code Starts for mapping
                    if (IdWithOrderLines.containsKey(orderLineId)) {
                        OrderItem orderLine = IdWithOrderLines.get(orderLineId);
                        for (String orderLineField : fieldSet) {
                            if (parameters.containsKey(orderLineField) && null != orderLineField) {
                                 if (!serialNumberFields.contains(orderLineField)) {
                                    orderItemIdWithAsset.get(orderLineId).put(parameters.get(orderLineField),orderLine.get(orderLineField));    
                                }
                                else{
                                    if (orderItemIdWithAsset.containsKey(orderLineId)
                                        && IdWithOrderLines.containsKey(orderLineId)
                                       ){
                                           if(orderItemIdWithAsset.get(orderLineId).SerialNumber == null){
                                                if(orderLine.OSIS_Serial_Number__c.length() < 80) {
                                                orderItemIdWithAsset.get(orderLineId).SerialNumber =
                                                    orderLine.OSIS_Serial_Number__c;
                                                }
                                                else{
                                                orderItemIdWithAsset.get(orderLineId).SerialNumber =
                                                    orderLine.OSIS_Serial_Number__c.left(80);
                                                }
                                                if(orderLine.OSIS_Serial_Number__c.length() < 80) {
                                                orderItemIdWithAsset.get(orderLineId).OSIS_Equipment_Number__c =
                                                    orderLine.OSIS_Equipment_Number__c;
                                                }
                                                else{
                                                orderItemIdWithAsset.get(orderLineId).OSIS_Equipment_Number__c =
                                                    orderLine.OSIS_Equipment_Number__c.left(80);
                                                }
                                           }
                                       }
                                }
                            }
                        }
                        for (String orderField : defaultValues.keyset()) {
                            if (defaultValues.containsKey(orderField)) {
                                orderItemIdWithAsset.get(orderLineId).put(orderField,defaultValues.get(orderField));  
                            }
                        }
                    }
                }
            }
           
        }
        catch(Exception exc) {
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
   
/**********************************************************************************************
* Method Name:updateAssetStatus
* Purpose:  This method is used to update Asset Status for sold to account.
This is used for transfer/reassignment flow where we need to change the status of sold to account's assets to Sold.
***********************************************************************************************/
    public static void updateAssetStatus(List<Asset> newAssetList) {
        Set<Id> sellerAssetIds = new Set<Id>();
        List<Asset> assetRecList = new List<Asset>();
        for(Asset aRec : [SELECT Id,SBQQ__QuoteLine__r.OSIS_Sold_Asset__c FROM Asset WHERE Id IN: newAssetList]){
            if(aRec.SBQQ__QuoteLine__r.OSIS_Sold_Asset__c != null)
                sellerAssetIds.add(aRec.SBQQ__QuoteLine__r.OSIS_Sold_Asset__c);
        }
        if(sellerAssetIds != null) {
            for(Id recId : sellerAssetIds){
                Asset aRec = new Asset(Id = recId);
                aRec.Status = Label.OSIS_CPQ_Transferred_Reassigned;
                assetRecList.add(aRec);
            }
        }
       
        if(!assetRecList.isEmpty()){
            update assetRecList;
        }
    }

/**********************************************************************************************
* Method Name: changeCombineKeyOnRevisedAsset
* Purpose:  This method is used to populate combine key field on revised asset.
This will help to merge assets belonging to same Product Code to single quote line.
***********************************************************************************************/    
    public static void changeCombineKeyOnRevisedAsset(List<Asset> newAssetList,Map<Id,Asset>oldAssetMap){
        try{
            Map<String,String>revisedAssetIdWithCurrentCombineKey = new Map<String,String>();
            for (Asset currentAsset : newAssetList) {
                if (oldAssetMap.get(currentAsset.Id).SBQQ__CombineKey__c !=
                    currentAsset.SBQQ__CombineKey__c) {
                    revisedAssetIdWithCurrentCombineKey.put(currentAsset.SBQQ__RevisedAsset__c,
                        currentAsset.SBQQ__CombineKey__c);
                }
            }
            if (revisedAssetIdWithCurrentCombineKey.size() > 0) {
                List<Asset>revisedAssetsToBeUpdated = new List<Asset>();
                for (Asset revisedAsset : [SELECT Id,SBQQ__CombineKey__c FROM
                                            Asset WHERE
                                            Id IN : revisedAssetIdWithCurrentCombineKey.keyset() ] ) {
                    if (revisedAssetIdWithCurrentCombineKey.containsKey(revisedAsset.Id)) {
                        Asset originalRevisedAsset = new Asset();
                        originalRevisedAsset.Id = revisedAsset.Id;
                        originalRevisedAsset.SBQQ__CombineKey__c = revisedAssetIdWithCurrentCombineKey.get(revisedAsset.Id);
                        revisedAssetsToBeUpdated.add(originalRevisedAsset);
                    }
                }
                if (revisedAssetsToBeUpdated.size() > 0) {
                    UPDATE revisedAssetsToBeUpdated;
                }
            }
        }
        catch(Exception exc) {
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }

/**********************************************************************************************
* Method Name: createAssetsAndSubscriptionsForPiDevClub
* Purpose:  This method is used create subscriptions and assets for every unit of PI-DEVCLUB.
Eg: If the qty for PI-DEVCLUB is 10 then we need to create 10 subscriptions with qty 1
and corresponding assets for them
***********************************************************************************************/
    public static void createAssetsAndSubscriptionsForPiDevClub(List<Asset> newAssetList,Map<Id,Asset>oldAssetMap) {
        try {
            Set<Id>piDevClubAssets = new Set<Id>();
            Set<Id>piDevClubSubs = new Set<Id>();
            Map<Id,Decimal>pidevClubSubsMap  = new Map<Id,Decimal>();
            Map<String,String> subSerialNumberMap = new Map<String,String>();//added 12/6
            for (Asset currentAsset : newAssetList) {
            if ((oldAssetMap.get(currentAsset.Id).SBQQ__RequiredBySubscription__c != currentAsset.SBQQ__RequiredBySubscription__c &&
                    currentAsset.SBQQ__RequiredBySubscription__c != null &&
                    Label.OSIS_DevClub.equals(currentAsset.OSIS_Req_By_Subscription_Product_Code__c)
                   )) {
                       piDevClubAssets.add(currentAsset.Id);
                       piDevClubSubs.add(currentAsset.SBQQ__RequiredBySubscription__c);
                   }  
            }
           
            List<String> piDevClubSerialNumbers = new List<String>();
            List<String> piDevClubEquipmentNumbers = new List<String>();
            Map<Id,OrderItem> mapOLI = new Map<Id,OrderItem>();
            Map<Id,OrderItem> mapParentQL = new Map<Id,OrderItem>();
            string OrderId;
           
            Map<String,List<SBQQ__Subscription__c>>mainSubscriptionWithAddOnSubs = new Map<String,List<SBQQ__Subscription__c>>();
            if (piDevClubSubs.size() > 0) {
                List<SBQQ__Subscription__c>subscriptions = [SELECT id,OSIS_Net_Quantity__c,SBQQ__Quantity__c,
                                                            SBQQ__Account__c,SBQQ__Discount__c,
                                                            SBQQ__AdditionalDiscountAmount__c,
                                                            SBQQ__Contract__c,SBQQ__CustomerPrice__c,SBQQ__ListPrice__c,
                                                            SBQQ__NetPrice__c,SBQQ__Bundle__c,SBQQ__ProductOption__c,
                                                            SBQQ__Product__c,SBQQ__PricingMethod__c,SBQQ__Dimension__c,
                                                            SBQQ__QuoteLine__c,SBQQ__RegularPrice__c,
                                                            SBQQ__RenewedDate__c,SBQQ__RequiredById__c,
                                                            SBQQ__SubscriptionEndDate__c,
                                                            SBQQ__SubscriptionStartDate__c,
                                                            SBQQ__OrderProduct__c,OSIS_Default_Clickwrap__c,
                                                            SBQQ__QuoteLine__r.SBQQ__Quote__r.OSIS_Sold_To_Account__c,
                                                            OSIS_Serial_Number__c,
                                                            OSIS_Equipment_Number__c,
                                                            SBQQ__OrderProduct__r.OSIS_Serial_Number__c,
                                                            SBQQ__OrderProduct__r.OSIS_Equipment_Number__c
                                                            FROM
                                                            SBQQ__Subscription__c WHERE Id IN : piDevClubSubs ] ;
                List<SBQQ__Subscription__c>updatedSubscriptions = new List<SBQQ__Subscription__c>();
                List<SBQQ__Subscription__c>updateParentSubscriptions = new List<SBQQ__Subscription__c>();
               
                //Below code is to identify the Serial/Equipment number in case of subscription triggers fires after asset trigger (No control on Triggers order of execution).
                set<id> setQLIIds = new set<id>();
                for (SBQQ__Subscription__c pidevClubSub : subscriptions) {
                    if(pidevClubSub.OSIS_Equipment_Number__c == null){
                        setQLIIds.add(pidevClubSub.SBQQ__QuoteLine__c);
                    }
                }
               
                if(setQLIIds.size()>0){
                    //Add SBQQ__QuoteLine__r.parentquoteline
                    List<OrderItem> lstOLI= [select OrderId, OSIS_Serial_Number__c,OSIS_Equipment_Number__c,SBQQ__QuoteLine__c,SBQQ__QuoteLine__r.SBQQ__RequiredBy__c from OrderItem where SBQQ__QuoteLine__c IN : setQLIIds];
                    for(OrderItem OLI: lstOLI){
                        OrderId = OLI.OrderId;
                    mapOLI.put(OLI.SBQQ__QuoteLine__c,OLI);
                        if(OLI.SBQQ__QuoteLine__r.SBQQ__RequiredBy__c !=null){
                        mapParentQL.put(OLI.SBQQ__QuoteLine__r.SBQQ__RequiredBy__c,OLI);    //
                        }
                    }
                }
               
List<SBQQ__Subscription__c>  parentSubs = [SELECT ID FROM SBQQ__Subscription__c WHERE SBQQ__Quoteline__c IN:mapParentQL.keySet()];//where QLI=  mapParentQL            
                for (SBQQ__Subscription__c pidevClubSub : subscriptions) {
                    pidevClubSubsMap.put(pidevClubSub.Id,pidevClubSub.OSIS_Net_Quantity__c);
                    // This logic will run for one record
                    if (pidevClubSub.SBQQ__Quantity__c !=null && pidevClubSub.SBQQ__Quantity__c > 0) {
                        Integer piDevClubSerialNumber = 0;
                       
                        // Below code is to identify the Serial/Equipment number in case of subscription triggers fires after asset trigger (No control on Triggers order of execution).
                        OrderItem OLI = mapOLI.get(pidevClubSub.SBQQ__QuoteLine__c);
                        if(pidevClubSub.OSIS_Equipment_Number__c == null && OLI != null){
                            piDevClubSerialNumbers = OLI.OSIS_Serial_Number__c.split(',');
                            piDevClubEquipmentNumbers = OLI.OSIS_Equipment_Number__c.split(',');

                        }else{
                            piDevClubSerialNumbers = pidevClubSub.OSIS_Serial_Number__c.split(',');
                            piDevClubEquipmentNumbers = pidevClubSub.OSIS_Equipment_Number__c.split(',');
                        }
                       
                        for (Integer i = 1 ; i < pidevClubSub.SBQQ__Quantity__c ; i++ ) {
                            SBQQ__Subscription__c bifurcatedPiDevClub = new SBQQ__Subscription__c();
                            bifurcatedPiDevClub = pidevClubSub.clone(false,true,false,false);
                            String piDevClubChildEquipmentNumber = '';
                            String piDevClubChildSerialNumber = '';
                            if (piDevClubSerialNumbers.size() > i) {
                                piDevClubChildEquipmentNumber = piDevClubEquipmentNumbers[i];
                                piDevClubChildSerialNumber = piDevClubSerialNumbers[i];
                            }
                           
                            bifurcatedPiDevClub.SBQQ__Quantity__c = 1;
                            bifurcatedPiDevClub.OSIS_Net_Quantity__c = 1;
                            bifurcatedPiDevClub.SBQQ__Number__c = i;
                            bifurcatedPiDevClub.OSIS_Serial_Number__c = piDevClubChildSerialNumber;
                            bifurcatedPiDevClub.OSIS_Equipment_Number__c = piDevClubChildEquipmentNumber;
                            bifurcatedPiDevClub.SBQQ__RenewalQuantity__c = 1;
                            if(parentSubs.size() > 0 ){
                                bifurcatedPiDevClub.SBQQ__RequiredById__c = parentSubs[0].Id;//
                            }
                            //sbqq__RequiredBY
                           
                            bifurcatedPiDevClub.OSIS_Sold_To__c =pidevClubSub.SBQQ__QuoteLine__r.SBQQ__Quote__r.OSIS_Sold_To_Account__c;
                            updatedSubscriptions.add(bifurcatedPiDevClub);
                            if (mainSubscriptionWithAddOnSubs.containsKey(pidevClubSub.Id)) {
                                mainSubscriptionWithAddOnSubs.get(pidevClubSub.Id).add(bifurcatedPiDevClub);
                            }
                            else{
                                List<SBQQ__Subscription__c> addOnSubs = new List<SBQQ__Subscription__c>{bifurcatedPiDevClub};
                                    mainSubscriptionWithAddOnSubs.put(pidevClubSub.Id,addOnSubs);
                            }
                        }  
                       
                        //Below code will update parent subscription where quantity is more than 1
                        if(pidevClubSub.SBQQ__Quantity__c >1){
                            pidevClubSub.SBQQ__Quantity__c = 1;
                            pidevClubSub.OSIS_Net_Quantity__c = 1;
                            pidevClubSub.OSIS_Sold_To__c = pidevClubSub.SBQQ__QuoteLine__r.SBQQ__Quote__r.OSIS_Sold_To_Account__c;
                            //updatedSubscriptions.add(pidevClubSub);
                           
                            SBQQ__Subscription__c updateParent = new SBQQ__Subscription__c();
                            updateParent.SBQQ__Quantity__c = 1;
                            updateParent.Id = pidevClubSub.Id;
                           
                            String piDevClubEquipmentNumberMain =   '';
                            String piDevClubSerialNumberMain = '';
                            if (piDevClubSerialNumbers.size() > 0 ) {
                                piDevClubEquipmentNumberMain = piDevClubEquipmentNumbers[0];
                                piDevClubSerialNumberMain = piDevClubSerialNumbers[0];
                            }                          
                            updateParent.OSIS_Equipment_Number__c = piDevClubEquipmentNumberMain;
                            updateParent.OSIS_Serial_Number__c = piDevClubSerialNumberMain;
                            updateParent.SBQQ__RenewalQuantity__c = 1;
                            updateParentSubscriptions.add(updateParent);
                        }
                    }
                }
                if (updatedSubscriptions.size() > 0) {
                    SBQQ.TriggerControl.disable();
                    UPSERT updatedSubscriptions;
                }
                if (updateParentSubscriptions.size() > 0) {
                    updateQytForParentSubscription(JSON.serialize(updateParentSubscriptions));  
                }
                /*********************/
                Set<Id> subIds = new Set<Id>();
                for(SBQQ__Subscription__c subRec : updatedSubscriptions){
                subIds.add(subRec.Id);    
                }
                subIds.add(updateParentSubscriptions[0].Id);
                for(SBQQ__Subscription__c subRec : [SELECT Id,OSIS_Equipment_Number__c,OSIS_Serial_Number__c FROM SBQQ__Subscription__c
                                                    WHERE ID IN :subIds]){
                                                       
                subSerialNumberMap.put(subRec.Id,subRec.OSIS_Serial_Number__c);
                }
                /************/
            }
           
            if (piDevClubAssets.size() > 0) {
                List<Asset >assets = [SELECT id,OSIS_Net_Quantity__c,Quantity,SBQQ__RequiredBySubscription__c,
                                      SBQQ__RequiredByProduct__c,SBQQ__RequiredByAsset__c,Product2.ProductCode,
                                      SBQQ__RootAsset__c,ParentId,OSIS_License_Size__c,
                                      Name,AccountId,Status,SBQQ__ProductOption__c,
                                      SBQQ__QuoteLine__c,SBQQ__RegularPrice__c,SBQQ__PricingMethod__c,OSIS_Product_Code__c,
                                      SBQQ__OptionType__c,SBQQ__Number__c,SBQQ__ListPrice__c,SBQQ__Bundled__c,
                                      SBQQ__Discount__c,SBQQ__AdditionalDiscountAmount__c,OSIS_Default_Clickwrap__c,
                                      SBQQ__SubscriptionEndDate__c,SBQQ__SubscriptionStartDate__c,
                                      Product2Id,SBQQ__OrderProduct__c,OSIS_Equipment_Number__c
                                      FROM
                                      Asset  WHERE Id IN : piDevClubAssets ]  ;
                List<Asset>updatedAssets = new List<Asset>();
               
                //Below code is used to populate OrderProduct field in case of Subscription/Asset Trigger sequence issue - Added by Perumal
                mapOLI = new Map<Id,OrderItem>();
                if(OrderId != null){
                    List<OrderItem> lstOLI= [select OrderId, SBQQ__QuoteLine__c,OSIS_For_License__c  from OrderItem where OrderId =: OrderId];
                    for(OrderItem OLI: lstOLI){
                        mapOLI.put(OLI.SBQQ__QuoteLine__c,OLI);
                    }
                }
                             
                for (Asset pidevClubAsset : assets) {
                    // This logic will run for one record
                    if (pidevClubSubsMap.containsKey(pidevClubAsset.SBQQ__RequiredBySubscription__c) &&
                        pidevClubSubsMap.get(pidevClubAsset.SBQQ__RequiredBySubscription__c) != null &&
                        pidevClubSubsMap.get(pidevClubAsset.SBQQ__RequiredBySubscription__c) > 0) {
                            for (Integer i= 1 ; i < pidevClubSubsMap.get(pidevClubAsset.SBQQ__RequiredBySubscription__c) ; i++ ) {
                                Asset bifurcatedPiDevClubAsset = new Asset();
                                bifurcatedPiDevClubAsset = pidevClubAsset.clone(false,true,false,false);
                                bifurcatedPiDevClubAsset.Quantity = 1;
                                bifurcatedPiDevClubAsset.OSIS_Net_Quantity__c = 1;
                               
                                String piDevClubEquipmentNumber =  '';
                                String piDevClubSerialNumber =  '';
                                if (piDevClubSerialNumbers.size() > i) {
                                    piDevClubEquipmentNumber = piDevClubEquipmentNumbers[i];
                                    piDevClubSerialNumber = piDevClubSerialNumbers[i];
                                }
                               
                                bifurcatedPiDevClubAsset.SerialNumber = piDevClubEquipmentNumber;
                                bifurcatedPiDevClubAsset.OSIS_Equipment_Number__c = piDevClubEquipmentNumber;
                                bifurcatedPiDevClubAsset.OSIS_License_Size__c = '1';
                                integer j = i-1;
                                if (mainSubscriptionWithAddOnSubs.containsKey(pidevClubAsset.SBQQ__RequiredBySubscription__c) &&
                                    mainSubscriptionWithAddOnSubs.get(pidevClubAsset.SBQQ__RequiredBySubscription__c).size() > j) {
                                        bifurcatedPiDevClubAsset.SBQQ__RequiredBySubscription__c =
                                            mainSubscriptionWithAddOnSubs.get(pidevClubAsset.SBQQ__RequiredBySubscription__c)[j].id;

                                }
                                //Population of OrderProduct field in case of Subscription/Asset Trigger sequence issue - Added by Perumal
                                if(pidevClubAsset.SBQQ__OrderProduct__c == null
                                   && pidevClubAsset.SBQQ__QuoteLine__c != null
                                   && mapOLI.get(pidevClubAsset.SBQQ__QuoteLine__c) != null){
                                       bifurcatedPiDevClubAsset.SBQQ__OrderProduct__c = mapOLI.get(pidevClubAsset.SBQQ__QuoteLine__c).Id;
                                       bifurcatedPiDevClubAsset.OSIS_For_License__c = mapOLI.get(pidevClubAsset.SBQQ__QuoteLine__c).OSIS_For_License__c;
                                }
                               updatedAssets.add(bifurcatedPiDevClubAsset);
                            }  
                            String piDevClubEquipmentNumberMain =   '';
                            String piDevClubSerialNumberMain =   '';
                            if (piDevClubSerialNumbers.size() > 0 ) {
                                piDevClubEquipmentNumberMain = piDevClubEquipmentNumbers[0];
                                piDevClubSerialNumberMain = piDevClubSerialNumbers[0];
                            }
                            pidevClubAsset.SerialNumber = piDevClubEquipmentNumberMain;
                            pidevClubAsset.OSIS_Equipment_Number__c = piDevClubEquipmentNumberMain;
                            pidevClubAsset.Quantity = 1;
                            pidevClubAsset.OSIS_Net_Quantity__c = 1;
                            pidevClubAsset.OSIS_License_Size__c = '1';
                            updatedAssets.add(pidevClubAsset);
                        }
                }
               
                for(Asset assetRec : updatedAssets){
                if(subSerialNumberMap.containsKey(assetRec.SBQQ__RequiredBySubscription__c) &&
                      subSerialNumberMap.get(assetRec.SBQQ__RequiredBySubscription__c)!=null){
                    assetRec.SerialNumber = subSerialNumberMap.get(assetRec.SBQQ__RequiredBySubscription__c);
                        assetRec.OSIS_Equipment_Number__c = subSerialNumberMap.get(assetRec.SBQQ__RequiredBySubscription__c);  
                }    
                }
               
                if (updatedAssets.size() > 0) {
                    SBQQ.TriggerControl.disable();
                    isRecurrsive = true;
                    UPSERT updatedAssets;
                }
            }
        }
        catch(Exception exc) {
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }

/**********************************************************************************************
* Method Name: updateQytForParentSubscription
* Purpose:  This method is update parent subscription qty to 1 for  PI-Dev CLUB.
***********************************************************************************************/
    @future
    public static void updateQytForParentSubscription(String updateSubscriptions){
        try{
            List<SBQQ__Subscription__c> updateParentSubscriptions = (List<SBQQ__Subscription__c>)
                                                                JSON.deserialize(updateSubscriptions,List<SBQQ__Subscription__c>.class);
            OSIS_SubscriptionTriggerHelper.runSubscriptionTrigger =false;  
            UPDATE updateParentSubscriptions;
        }
        catch(Exception exc) {
            OSIS_CreateErrorLog.logException('OSIS_AssetTriggerHelper',exc,false);
        }
    }
   
/**********************************************************************************************
* Method Name: populateRequiredByAssets
* Purpose:  This method will populate missing field "RequiredByAsset" for child assets in
case of Subscription trigger fires after Asset triggers.
***********************************************************************************************/
    public static void populateRequiredByAssets(set<id> setQuoteIds){ //setAssetIds){
        Map<String,String> mapOrderProduct = new Map<String,String>();
        Map<String,String> mapRequiredByAsset = new Map<String,String>();
        set<id> setChildAssetProd = new set<id>();
        Boolean IsAssetToBeUpdated = false;
       
        List<Asset> updateAssets = new List<Asset>();
        List<Asset> lstAsset =[select id,product2Id, product2.productcode, SerialNumber, SBQQ__RequiredbyID__c,SBQQ__RequiredByAsset__c,
        SBQQ__RequiredBySubscription__c, SBQQ__OrderProduct__c, SBQQ__ProductOption__c, SBQQ__ProductOption__r.SBQQ__ConfiguredSKU__c
        //from asset where ID IN: setAssetIds ];
        from asset where SBQQ__QuoteLine__r.SBQQ__Quote__c IN: setQuoteIds ];
       
        for(Asset A: lstAsset){
            if(A.SBQQ__RequiredByAsset__c != null){
                setChildAssetProd.add(A.Product2Id);
            }
           
            if(A.product2Id == Label.OSIS_CC_PVS && A.SerialNumber != null){
                mapRequiredByAsset.put(A.SerialNumber, A.Id);
            }
        }
       
        for(Asset A: lstAsset){
            if(A.SBQQ__ProductOption__r.SBQQ__ConfiguredSKU__c == Label.OSIS_CC_PVS
               && A.SerialNumber != null
               && A.SBQQ__RequiredByAsset__c == null
               && mapRequiredByAsset.get(A.SerialNumber) != null
               && setChildAssetProd.contains(A.Product2Id)){
                   A.SBQQ__RequiredByAsset__c = mapRequiredByAsset.get(A.SerialNumber);
                   IsAssetToBeUpdated = true;
               }
            //updateAssets.add(A);
        }
        SBQQ.TriggerControl.disable();
        isRecurrsive = true;
        //upsert updateAssets;
        if(IsAssetToBeUpdated){
            upsert lstAsset;
        }
        SBQQ.TriggerControl.enable();
    }
}
