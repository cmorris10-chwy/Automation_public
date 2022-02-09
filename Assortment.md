# Inventory Assortment Knowledge Document

By: Cory Morris

## Purpose

This document is meant to give detail into how assortment works as of 1/25/2022 at Chewy. It will discuss what inputs Assortment needs, where that data resides, both assortment processes (daily, monthly), and how assortment is used by its customers. The inventory assortment process discussed in this document does not include Pharmacy  Fulfillment Centers (FC) or Pharmacy items.

## Inventory Assortment

### Assortment Processes

Inventory Assortment has two processes: *Daily* and *Monthly*. Both processes do not include discontinued items. A discontinued item will remain assorted to the FCs it was assorted to on the day it was discontinued. This is becuase there is a requirement for ORS' 3-year demand file process which needs an item's last known assortment as an input. The scope of the *Daily* process includes all items with the *New Item Launch* and any any item-locations that had an assortment override add/removed/updated. This process runs every evening at 8PM EST. *New Items* are identifiable by conditions: `mercury.item_attr.attr_cd = "NEW_ITEM"` and `mercury.item_attr.attr_val = true`. On average **XX** item-locations are re-assorted each day from this process because of changing velocity codes and/or UOM status being updated. Having this process ran on daily cadence is inefficient as demand will not be allocated to the net new item-locations until the following Monday - when demand realignment batch run occurs. Since this process is currently *Daily*,the *Daily* process should only include item-locations which assort items with any override changes and/or cartonization flag changes. This will ensure that less computational resources are needed and will keep our assortment correct on a daily basis with the help of the [Process Gap Fixes](#process-gap-fixes-short-term) explained later in this document.

The *Monthly* process runs the first Saturday of each month. The scope includes all active, non-Pharmacy items within Chewy's catalog - including *New Items*. It also processes all assortment overrides so that any changes are relfected within the assortment. Like the *Daily* process, after the *Monthly* process updates the assortment, the net new assorted item-locations will have to wait until Monday's demand realignment process before it receives a forecast.

Knowing how these processes work the batch cadences should be updated. The *Daily* process should only include items with assortment override changes or `cartonization_flag` changes. The *Monthly* process should move to a weekly cadence (each Sunday) as this will ensure that the full catalog is re-assorted based on latest item attributes while making sure that once they are assorted that demand will be allocated the following day (Monday). ***Note***: An analysis is in-progress to analyze the assortment effects from both of the current processes.

### Assortment Configurations

Both processes assort an item based configurations that have been set by Inventory Optimization. The current configurations have been in use since 2019 I believe. The configurations dictate which network velocity codes and UOM statues are eligible at each FC. See [Appendix ?](#appendix-current-assortment-configurations) for the assortment configurations as of 1/25/2022. Velocity codes with a `+` indicate UOM items.

UOM represents an item that ships in the same packaging that the vendor shipped the item to Chewy. These items cannot fit in a Chewy container - current max container is C300 - as their dimensions are too large. HJ owns maintaining the `aad.t_item_uom.cartonization_flag` which identifies an item as UOM if the max(L,W,H) > 25" or if the item's weight is greater than 40 lbs. If the item has not been sampled by a [Sample Center](#sample-center-information) then the dimensions loaded by the sourcing manager during item setup will be used. This logic is subject to change but this is the current implementation.

You can identify UOM items using the following SQL query. This is the proper way to identify UOM items as stated by FC Ops.

```{SQL}
select item_number
    , cartonization_flag
from aad.t_item_uom
where wh_id='AVP1'
```

### Assortment Overrides

Assortment Overrides give us the necessary functionality to address use cases which the current assortment configuration does not account for. This is because the current processes do not account for other item attributes outside of `cartonization_flag` and `product_abc_code`. For example, overrides are currently being used to address the below use cases:

1. FreshPet
     - This brand's items require refrigeration/freezers but the business wants to store these items within our Retail FCs instead of CFF/WFF. To allow these items to be assorted correctly into AVP1, CFC1, and WFC2 we utilize overrides as they are classifed as UOM items and such will never meet the configurations for these FCs. The overrides force these items into the assortment.
     - Users need to keep FreshPet items in mind when ensuring that UOM items are not assorted to non-UOM FCs as these items will show as ineligible but are actually correctly assorted

2. Cartonization Flag Updates

     - There are times when an item's `aad.t_item_uom.cartonization_flag` will change from False (UOM) to True (non-UOM). The two reasons for this are:

          - FC employee updates dimensions in HJ. Reason codes are not used when changes occur so the root cause is unknown. From discussions with Quality Assurance (QA) warehouse employees will change dimensions whenever they need to force a putaway as HJ won't allow an item with larger dimension to be stored in smaller bins - even though the employee can clearly see it will fit. QA is currently implementing access restrictions for making changes in HJ. The ETA is end of 2022.
          - The item was never sampled by a Sample Center. The reason is because not all new items had a sample order created. Due to long lead-teams Private Label (PL) cannot get sample items sent ahead of the initial shipment. Therefore, PL purchases the initial order containing any new items into the appropriate [Sample Center](#sample-center-information). This causes PL new items to get initially assorted to ineligible FCs which allows replenishment. For example, if an item is initially a non-UOM item it will be assorted to all FCs. Then when the PO arrives, the item is sampled and the `cartonization_flag` updates to FALSE. This will cause UOM items to be assorted to our non-UOM only buildings which cause issues for Operations.

     - In an ideal world a new item to the catalog will get sampled in a Sample Center prior to the item completing the New Item Setup process. Since getting a new item sampled is currently not a requirement in the New Item setup process, this will continue to happen.
     - When this flag changes, an item will not be re-assorted until the next Daily run (if a New Item) or next Monthly run. This is why I have implemented a [manual process](#cartonization-flag-changes) so that these items get re-assorted immediately. I have a [SCP-5123](https://chewyinc.atlassian.net/browse/SCP-5123) open to automate this process.

3. Zero Forecast Item

     - This is a legacy process that is used to limit items that have a very low forecast to two FCs until sales materialize. FCs: AVP1, PHX1
     - This process is only being used by 1 or 2 Demand Planners since most of the DPs, at time of implementation, have left the company and new planners did not learn the process.
     - This is discussed in more detail [below](#new-item-assortment).

4. FC Constraints

     - **non-UOM only:** These FCs cannot handle any UOM items due to not having the capabilities to receive, store, and ship them. All second generation (2G) buildings are non-UOM in addition to EFC3. This constraint is being enforced via the *Daily* and/or *Monthly* batches. As discussed in #2 above, overrides are used to immediately de-assort ineligible items to ensure replenishment is killed.
     - **Tiered Racking:** FCs that do not have the necessary racking needed to store overhang pallets. These items are being tracked [here](L:\Cory Morris\Assortment Overrides\Alerts). This list is being updated based on items that Ops has told me cannot be assorted. I believe that these items are anything that have a `54` within the item description.
     - **HAZMAT**: This is currently not a constraint for any active FC, but could be for future FCs initially.
     - **AutoStore (AS)**: This is currently not a hard constraint for the FCs with AutoStore functionality. 2G buildings have Autostore capabilities. In the future Operations might want to adjust the proportion of AS vs. non-AS items.
     - **AutoPack (AP)**: This is currently not a hard constraint for any FC. AutoPack is at MDT1 currently but MDT1 also handles non-UOM items that ship as singles primarily. This is why it is not a hard constraint. Like AutoStore, Operations might want to adjust the assortment proportion for these items. The FC wants AP items to take priority over non-AP if the FC were to ever become capacity constrained.

5. Adhoc Assortment Requests

     - CFC1 + WFC2

         - Exclude all *NEW_ITEM*s from CFC1 and WFC2.
         - This is to eleviate capacity issues as CFC1 and WFC2 have an Maximum Efficient SKU Capacity (MESC) of 45k.

Assortment overrides can be found using query:

``` {SQL}
select *
from mercury.assortment_override
```

#### Override Reason Codes

On January 1, 2022, before the monthly batch run, all current overrides were evaluated to identify overrides which were no longer needed and were subsequently removed from the `mercury.assortment_override` table. The remaining overrides have been given a reason code so that an automated clean-up processes can be implemented utilizing these reason codes. Since this new process is not implemented yet, I run a manual process the Friday before each Monthly batch run. The manual process is as follows:

1. Disposition all current assorment overrides using script in [Appendix 3](#appendix-2-override-disposition-script)
2. Remove all dispositions that "KEEP:..."
3. With only the overrides to be removed, run this [script](https://github.com/cmorris10-chwy/Automation/blob/main/Remove_Overrides/run/generate_override_import_file.py)
     - The output of this script will give you the necessary files (in correct format) to be uploaded to Mercury through the UI. These files will only include the overrides to remain (KEEP) or to be updated

4. Delete all current overrides in Mercury for the FC you are uploading
5. Upload FC's import file using the [Mercury UI](https://mercury-ui.scff.prd.chewy.com/)
     - Follow path: *Demand Planning* > *Assortments* > *select FC from dropdown* > *Search*

Whenever new overrides are added or existing ones are updated, I insert the item-location and reason code into this [tracker](L:\Cory Morris\Assortment Overrides\Override Tracking\Item Assortment Override Reason Code Tracker v1.xlsx).

### Assortment of One-Time-Buy Items (OTB)

OTB items are assorted using the same processes. Merchandising (Merch) has verticals which own managing OTB items. OTB items usally get a future launch date applied during item setup. This launch date is used so that replenishment and forecasts do not start until a lead-time before the launch date. This launch date represents when the item should be available to sell within the FC.

OTB items are supposed to be true, one time buy purchases where we buy a bulk quantity pre-season and when it sells out then the item is discontinued. This is not the case though as there are times when an item outperforms initial sales projections and we want to re-stock. In these cases, if there is available inventory at the supplier, then Merch will place an additional order. These are not true OTB items and therefore should be treated differently.

**GAP:** Each Merch team handles OTB items differently. The ideal process is being done by one team already (forget the name). She has setup a process where OTB items are truly ordered one time and when the item goes network out-of-stock (OOS), e.g., OH = 0, the item is discontinued so that replenishment ceases and the item falls out of assortment scope. By discontinuing the item immediately, the item will not be counted towards the buildings MESC which opens capacity for other items to be assorted there. This is especially important for our smaller FCs such as CFC1 and WFC2. Furthermore, SP does not order inventory for all assorted OTB item-locations. If this is the case, we need to understand why that is (i.e., limited supplier inventory?), and then potentially create logic to assort OTB items differently.

### Assortment of Replacement Items

Replacement items are items that use another item's past sales history to generate a forecast - **Still awaiting confirmation**. These items can be found using [this query](#appendix-replacment-items-query) within Mercury database. The Mercury assortment processes handle these items by assorting the `item_id` SKU based on the `replacement_item_id` SKU's velocity code.

**GAP:** There are instances where the `replacement_item_id` SKU's velocity code is lower than the `item_id` SKU's velocity code. This does not make sense as sometimes these `replacment_item_id` SKU's are discontinued and thus have no materializing sales. This will make its velocity code go to `E` over time. If the `item_id` SKU is a higher velocity code then we are limiting its assortment because we are assorting it as an E-velocity item. This needs to change so that Demand Planning still has the ability to see the replacement items so they can use the past sales history. But have assortment use the `item_id` SKU's velocity code in cases where it's velocity code is higher than the replacement items or if the replacement item is discontinued.

### Assortment of De-bundle Items

De-bundle items are ones that ship from the vendor is a larger container and then are shipped to the customer as a unit of that larger container. For example, if a vendor case pack (VCP) is 15 over 3, then that indicates that we ship an inner of 5 units for each customer order as this item is sold in boxes of 5 units. For supply planning purposes we replenish at the parent item level and sell to the customer at the child item level. De-bundle items can be found using [this query](#appendix-de-bundle-item-query).

**GAP:** There are times when the parent or child items are not assorted to the same FC due to one of them either being discontinued - thus out of scope - or one has an ineligible velocity code or UOM status. This causes replenishment or order fulfillment gaps. In other words if the child item is not assorted to the same FC as the parent item, then its inventory at that FC will never sell down. On the other hand, if the parent item is not assorted to the same FC as the child item, then it will never have inventory OH as there will be no proposals for the parent item due to not being assorted.

### Checking Current Assortment

To check the current assortment item count by FC you can use [this query](#appendix-current-assortment-query).

### New FC Launch Assortment

The current process for managing assortment at new FCs is a shared responsibility across S&OP and Inventory Optimization with limited systematic support, resulting in a fragmented and manual process. In a steady state, established FC assortments are managed systematically through item attributes such as velocity code and cartonization flag to identify which items are assorted to which FC. The cartonization flag (aad.t_item_uom.cartonization_flag) represents whether an item ships in its own container (UOM) or a Chewy container (non-UOM). The flag is maintained by HighJump (HJ) by using the item dimensions loaded by Quality Assurance (QA/Sampling) during the new item process. If not sampled, then the product dimensions input by the Sourcing Managers would be the default. The velocity code represents the items network velocity (chewybi.products.product_abc_code). These two attributes are what the current automated assortment processes use to assort items to FCs. The other attributes are unusable as they currently are unidentifiable within our data, or our current processes do not account for them yet.

These attributes are:

1. AutoStore
     - This is for 2G buildings (AVP2, MCI1, RNO1 next). Items are unable to have dimensions that cannot fit into a 23”x15”x12” tote.
     - Unidentifiable within the data
2. Oversized Items
     - These items are UOM (carontization_flag=False) and they also are too large to fit on standard racking and require tiered racking.
     - There are FCs that currently do not have tiered racking and cannot handle these items.
          - FCs: CFC1, MCO1, WFC2
     - Unidentifiable within the data. Assortment logic doesn’t currently account for this either.
3. Item Singles
     - These are items that are purchased primarily by themselves.
          - MDT1 is only FC currently using theses.
     - Unidentifiable within the data and Assortment logic doesn’t currently account for this.
4. AutoPack
     - Items that cannot fit in the AutoPack machines
     - For MDT1 only
     - Has a flag, aad.t_item_uom.autopack_eligibile, within HJ.
     - This is not a strict restriction though as MDT1 can also process OB manually if the item cannot fit within the AutoPack machine. However, AutoPack eligible items are preferred to be assorted as they are more efficient.
     - The assortment processes do not account for this currently.

For all the above restrictions the assortment program manager (APM) runs ad-hoc processes to try to keep ineligible items from being assorted to an FC. The reason for these ad-hoc processes is that the Mercury assortment processes do not yet account for the new restrictions which allows these ineligible items to be assorted to wrong FCs. More importantly though is that all but AutoPack are not identifiable within the data (i.e., no field). To solve this problem, we need all new FC restrictions to be identifiable (i.e., have a field) within a database that Mercury can consume or have clear definitions on how to identify such items from existing fields in the data. An example of this would be for AutoStore items. We know that an item has to fit into a tote (23”x15”x12”) for it to be eligible for AutoStore. Having all parties align on the definition for identifying a restriction will reduce future assortment issues. For warehousing restrictions, we will use HJ as the source of truth and for all others we will need to create an attribute within Mercury (e.g., Item Singles) or create it based on the defined logic. Systems team should own updating implementations within Mercury. Mercury will also need to implement a new way for the APM to change the configurations of an FC(s) when needed. Mercury currently only allows velocity and UOM status configurations to be set. For new FCs these restrictions need to be know as far in advance as possible so that the new attribute fields can be created and implemented into our assortment processes before the FC is fully ramped and moved over to the automated processes. This needs to be owned and communicated by Ops  up front so that all subsequent assortment activities are executed correctly.

In the short term the APM will manage ad-hoc processes on a daily cadence to create/update assortment overrides to either add an item into an FC’s assortment or remove it from the assortment. This process works but it has a 1-day gap. This gap allows these items that have a proposal generate for that day, due to being assorted, can be ordered. This creates downstream issues such as the need for transfers or diversions. The one-day gap caused by the fact that assortment overrides input during the day will not take affect until the next day since the assortment processes run in the early morning. Therefore, these items will remain orderable until the following day when the item’s assortment is updated. Canceling of POs that are created with ineligible items is time consuming on Supply Planners and creates truck rounding issues due to purchase minimums no longer being met. The ideal solution for this gap is to build the restrictions into the data or logic into Mercury and include in the daily assortment process so that the assortment can be corrected before SO99 generates that day’s proposals.

While the assortment model may change in the future, away from velocity-UOM based, we will always need to account for FC constraints when modeling an ideal assortment. Therefore, having definitions for each restriction and be identifiable within the data will always be needed to ensure that the assortment does not cause unnecessary issues.

Another issue is that the current, automated assortment processes cannot be leveraged for new FCs as it would assort all items at once, thereby blowing up their inbound and not allowing the site to ramp. It would blow up the FC’s inbound processes since inventory for the thousands of newly assorted items would start showing up on day 1 and the FC will not be prepared for that. As such SNOP works with Ops to create an assortment ramp plan which seeks to achieve certain inbound, capacity, and outbound targets through the course of the FC ramp. To achieve this ramp, manual and static lists are loaded for the site by the Systems team (Piyush). This staggers the number of items assorted to the FC at any given time to allow the building to properly ramp up to full capacity. Our current systems (Mercury/SO99) do not have the ability to do a phased purchase plan which is why this inefficient approach is used currently.

he static nature of this ramp plan creates various pain points. For example, if an item’s cartonization status changes and is no longer deemed eligible to ship in a chewy container, but a new FC is not supposed to carry non-cartonizable items, there is no dynamic way to remove that item from the assortment. It requires a human to identify this change and manually exclude an item from the assortment. Similarly, as our catalog grows and we add new items, new items are not incorporated into the new FC’s assortment ramp plan and therefore not purchased. When new, high velocity items launch, a person must manually adjust the assortment to include that item. The inventory analytics team has created automated alerts to identify defects within new FC assortment ramps, however, it is still dependent on humans to manually intervene and adjust plans.

Another issue is the activation and maintenance of our supplier-location records for existing, active vendors. Mercury Supplier-Location records are used to determine which FCs each vendor is enabled to ship to. These records are owned by the Supply Planning team and should be kept up to date as it is a key factor of item orderability. If an item is assorted to the new FC but an active and eligible vendor for that item is not enabled to ship to the new FC, then SO99 will never create a proposal for that vendor-new FC pair – thus leading to incorrect ramp projections since inventory is not being ordered. Having a clear timeline for when SP needs to create Supplier-Location records for the new FC will ensure that the assorted items are systematically replenishable. If Distribution vendors need to be delayed until the FC is fully ramped, then this will be an added reason for why a “full ramp date” is needed – discussed below. SP currently manage this task, but the current process has missed enabling vendors in the past which leads to not generating proposals for assorted items.

Lastly, key dates for when assortment activities are needed during the new FC launch process need to be clearly identified and communicated. This is because a lot of the current assortment activities are manual and will require time to perform. Key dates that need to be determined are:

- When does the initial assortment list need to be created and who needs it? How often/when should this assortment list be updated?
- When does the first list need to be input so that purchasing can begin?
- Upon launch, when should the assortment list be updated to include new items and/or remove/add items based on the FC’s restrictions?
- When is the FC fully ramped and can be moved to the automated Mercury assortment processes from the curated/fixed assortment?

### New Item Assortment

Any item that is re-activated or is brand new to the catalog has to successfully move through the *New Item Launch* process before it can be assorted. MSS owns this process. The reason for the distinction between re-activated and new items is because they are damand forecast differently. Any reactivated item that has more than 90 days of actual sales will be forecast by SO99. Otherwise the item will use the demand forecast input by the demand planner (DP) within the Mercury UI. Furthermore, any re-activated item that is forecast by SO99 are the items that, if re-assorted in the daily process, will not get any demand allocated until the following Monday 3-year demand batch runs. Reactivated items using the Mercury forecast and brand new items will get a new forecast allocated immediately if re-assorted.

The last step of the *New Item Launch* process, before an item is eligible to be assorted, is owned by DP. Technical details for the *New Item Launch* process can be found [here](https://chewyinc.atlassian.net/wiki/spaces/SCS/pages/152306812/Mercury+-+New+Item+Launch+Page+and+Assortments). The planner will forecast the item and select the `Ready to Process` check box. The planner will also select the `Zero Forecast` check box if the item has a super low forecast and we want to limit the item to 2 FCs - AVP1, PHX1. This allows sales to materialize until a more accurate forecast is generated. This process was implemented in 2019 or 2020 but has not been utilized as of late.

DP currently forecasts new items on Fridays because they want to ensure that all new items get demand allocated once assorted. On Friday DP - currently Nihar - sends Piyush a list of new items to be assorted that week. The reason for the list is because of the [CFC1+WFC2 overrides](#assortment-overrides) items. These require overrides to be created as Mercury assortment logic does not support this currently. If the logic did support this, then a checked `Ready to Process` box would signal to Mercury that the item is eligible to be assorted as the `Zero Forecast` process is already supported.

Before the item is assorted, Mercury performs a check to ensure the item has an active SPA. If an item does not then Mercury will apply an `mercury.item_attr.attr_val` as `Missing BPA`. If an active SPA exists, then the item will be automatically assorted per the [Assortment Configurations](#assortment-configurations).

**Gap:** The `Zero Forecast` process is only being followed for 1 or 2 planners. The other planners are new to Chewy and were never taught. There is opportunity to improve this process by making it dynamic and having the assortment uupdated automatically as sales materialize and an accurate forecast is applied.

## Upstream Details

Items that are re-activated or new to the catalog are put through the *New Item Launch* process which Merch owns and tracks within Jira. MSS first creates the item or re-activates the item within Oracle. This includes creating active Supplier Purchase Agreements (SPA) and inputting all item information. Upon completion the Jira ticket is handed off to Supply Planning to create the initial sample order and to create supplier-location records within Mercury. Supplier-location records are one of the two requirements for an item to be orderable within SO99. The other requirement is an active SPA - see Appendix 1 for query to find active SPAs. In parrallel Demand Planning (DP) will begin creating an item forecast. When complete they set the *Ready to Process* flag to TRUE and then send an email to Systems team with a list of new items. The list is only so that we can manually create assortment overrides for to enforce two interim use cases. The first being that CFC1 and WFC2
are experiencing capacity issues and cannot take on more items. To help meet this need all D+E network velocity items that flow through the *New Item Launch* process are excluded from these locations' assortment. The second use case is for *Zero Forecast* items. Such items are ones that DP cannot accurately predict the item's forecast due to not enough data or similar items. These *Zero Forecast* items are initially only assorted to AVP1 and PHX1 - excluded at all other FCs.

## Downstream Details

In this section we will discuss how assortment impacts its customers: Demand Planning, Supply Planning, and Operations.

### Demand Planning

Assortment informs SO99 where to distribute the network forecast to. If an item is not assorted to an FC then that item-location will not be allocated a forecast. Before SO99 generates the forecasts, ORS generates a list of total sales for each item-location in the past 3 years. This file is generated every Monday and can be found within SQL Server table `LINEORDERS_FULL`. If an item has never shipped out of an FC, even if it is assorted there, the item will not receive a forecast. More details can be found [here](https://chewyinc.atlassian.net/wiki/spaces/SCS/pages/9498471/Historical+Sales). **This is a current gap**. Futhermore, with the way that allocation is set across the FCs, not all assorted items will receive a forecast. Ratna understands this topic the best, but the way I understand it is that if the allocation percentage has been met with the highest 60k items and there are a total of 65k items, then the tail-end 5k items will not be allocated a location forecast. When an item-location is not allocated a forecast then replenishment is disabled.

Demand Planning requires the assortment state to be archived and never change thereafter when an item is discontinued. The assortment within `chewybi.inventory_snapshot` should show the assortment for the item on the day that the item was discontinued. Since discontinued items are not in-scope for the assortment batches, their assortments will never change unless a user deletes the entire state or an override causes the item's assortment to change.

### Supply Planning (Replenishment)

For an item to be eligible for replenishment at an FC it first has to be assorted to the building. Once assorted there are two requirements that must be met to generate proposals for a supplier instead of self-transfers.

   1. Item must have an Active Supplier Purchase Agreement (SPA)
   2. Supplier for the active SPA must have an Enabled supplier-location record within Mercury for the FC you want to purchase into.

You can quickly see whether an item-location has an eligible vendor by using [this query](#appendix-eligible-vendor-for-item-location-query). If it is an FC vendor number then one of the above conditions is not met and you can use the below sections to troubleshoot. If the Supplier is a true vendor number then both conditions are being met and the vendor is who Mercury will purchase from.

#### Active SPA

To find whether an item has an active SPA or not you can either look within Oracle, Mercury Database, or a sandbox table that I update each morning with data from Mercury. Oracle is the source of truth for SPAs but Mercury is what drives Chewy’s purchasing and therefore, has additional filters to ensure the data it is receiving from Oracle is in fact good data. The main filter that Mercury applies to the SPAs retrieved from Oracle is based on the SPA order line number. If this value is 7 digits (1 million) or higher then Mercury assumes this is bad data and thus rejects the SPA. Such SPAs should be cleaned up by Merch Ops if found as these were generated by accident and a new SPA with a correct line number should be created.

An SPA is active if it meets all the three below conditions:

   1. Does not have a disable_rsn- Should be NULL
   2. Does NOT have an end_dt - Should be NULL
   3. The line_status is OPEN

Even though the above is the criteria for an active SPA, Mercury also considers the SPA's availability date. This date represents a date at which the item is available to be purchased from the vendor. If there is a future date then Mercury will not consider the Vendor in the primary vendor selection pool. This is one way to halt proposals for an item-vendor if the vendor is currently OOS and you do not want TG to consider the item when performing truck rounding.

To find active SPAs for an item within Mercury you can use [this query](#appendix-find-active-spa-within-mercury). **Note**, the issue with querying directly within Mercury DB is that you will not be able to join this data onto other tables that exist with Chewy DB such as the productstable. To get around this I import the Active SPAs from Mercury into a sandbox M-F (9am) which can be used. Here is the query to get the latest active SPAs.

```{SQL}
select vendor_distribution_method, vendor_name, spa.*
from sandbox_supply_chain.cmorris10_supplier_purchase_agreement spa
join chewybi.vendors v on spa.supplier_cd=v.vendor_number
where 1=1
        and item_id='123456'
```

#### Proposals

If the active SPA supplier has an Enabled supplier-location for the FC you are investigating then Mercury will consider that supplier when setting the primary vendor for the item-location as stated above. If their is a non-FC primary vendor and TG is not proposing then it is most likely due to there not being a forecast for the item at that FC. You can check whether there is a forecast for the item-location using [this query](#appendix-item-location-forecast).

If the item is forecast complete - ([use this query to check](#appendix-item-launch-and-forecast-complete-dates-query) - but the forecast is either null or zero then it might have a future launch date which will dictate when the forecast starts. The launch date specifies when the item will be published online. Proposals will generate 1 lead time before this date.

If a launch date exists you will see that the forecast will become non-zero at or just before the launch date. If the item does not have a launch date and the item-location forecast is zero or null then there is another issue. This issue is discussed [above](#demand-planning) regarding the building allocation percentages and tail-end items being left off.

#### Mercury Supplier-Locations

Once you have found one or more active SPAs for the item, you will need to confirm that the supplier/vendor for the active SPAs haven supplier-location records created within Mercury and that those records are “Enabled” for the FCs you want to purchase into.

You can check supplier-location records through the [Mercury UI](https://mercury.chewy.com/#/supplier-location) or within `chewybi.vendor_location` table using [this query](#appendix-supplier-location-query). If a supplier-location record has a vendor_status of anything but `Enabled` then the system considers that vendor as not being eligible to ship to that FC and thus will not be used for purchasing. Primary Vendor selection logic can be found [here](https://chewyinc.atlassian.net/wiki/spaces/SCS/pages/9498948/Primary+Vendor+Selection+includes+MOQ).

Supplier-location records are initially setup during the vendor setup process by the SP assistants. When the new vendor Jira ticket is assigned to the SP assistant they will input all of the provided lead time and purchasing min/max information. They will also select the `Active` checkbox which will set the `vendor_status` to `Enabled`. These records have not been maintained indicating their data might be out of date - **opportunity**. Another example is that SP uses these records to kill replenishment for a vendor for a period of time but fails to re-activate the supplier-location record later. Disabledsupplier-locations lead to assorted items being primary through transfer within SO99. This currently accounts for ~3k item-locations without inventory on-hand per this [Assortment Gap report]().

### FC Operations

## Sample Center Information

There are two FCs that handle sampling items - AVP1 and PHX1. AVP1 is for all National Brand and non-UOM items. PHX1 is for all Private Label and UOM items. This was agreed upon by Quality Assurance's Kyle Rickey.

## Process Gap Fixes: Short-term

### Cartonization Flag Changes

### Ineligible UOM Purchase Order Line Items

## Appendices

### Appendix: Find Active SPA within Mercury

There are three options. Users can serach for an active SPA directly within Oracle and Mercury user interfaces (UI) by item number or vendor number. The third way is to query them within Mercury. Mercury
has additional filters that are applied during the dataload process from Oracle. I don't know all of the filters but I do know that if an SPA line number is greater than 9 digits long then it will be considered
"invalid" and will not be transferred over. These 9 digit line numbers are due to a next number issue that occurs during bulk uploading.

I have found that the SPA tables within `chewybi` do not match the data within Mercury. To get around this I have implemented my own process which downloads the current active SPAs from Mercury using the below query and load them into a sandbox table: `sandbox_supply_chain.cmorris10_supplier_purchase_agreement`. I try to update this table daily at 8AM EST, but since it relies on me to run the code, the table does not get updated when I am on vacation or on the weekends. I have the goal to automate this in the future.

Run the below within Mercury DB connection

```sql
select current_date::date as snapshot_date
        , supplier_cd
        , item_id
        , start_dt
        , end_dt
        , supplier_uom_cd
        , supplier_unit_price
        , supplier_item_id
        , line_number
        , min_ord_qty
        , ord_multiple
        , availibility_dt
        , status
        , disable_rsn
        , create_dt
        , update_dt
        , line_status
from mercury.supplier_purchase_agreement spa
where spa.disable_rsn is NULL
        and spa.end_dt is NULL
        and spa.line_status = 'OPEN'
;
```

### Appendix: Current Assortment Configurations

*NOTE*: A `+` following a velocity indicates UOM.

|FC   | A   | B | C | D | E | A+ | B+ | C+ | D+ | E+ |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| AVP1 | X | X | X | X | X |   |   |   |   |   |
| AVP2 | X | X | X | X |   |   |   |   |   |   |
| CFC1 | X | X | X | X | X | X | X | X |   |   |
| CLT1 | X | X | X | X | X | X | X | X |   |   |
| CFF1 | X | X | X | X | X | X | X | X | X | X |
| DAY1 | X | X | X | X | X | X | X | X | X | X |
| DFW1 | X | X | X | X | X | X | X | X | X | X |
| EFC3 | X | X | X |   |   |   |   |   |   |   |
| MCI1 | X | X | X | X | X |   |   |   |   |   |
| MCO1 | X | X | X | X | X | X | X | X | X | X |
| MDT1 |   |   |   |   |   | X | X | X | X | X |
| PHX1 | X | X | X | X | X | X | X | X |   |   |
| RNO1 | X | X | X | X | X |   |   |   |   |   |
| WFC2 | X | X | X | X | X | X | X | X | X | X |
| WFF2 | X | X | X | X | X | X | X | X | X | X |

### Appendix: AutoStore Logic

```{Python}
def autostore_eligible(length, width, height, unit_weight, daily_forecast, category):
    # Logic is based on Autostore Flow diagram from Matthew Jurado.
    # This is not a hard requirement for assortment
    tote_max = 23
    tote_mid = 15
    tote_min = 12
    tote_vol = 23*15*12

    item_volume = length*width*height
    if length > tote_min and width > tote_min and height > tote_min:
        decision = False
    elif (length > tote_mid and width > tote_mid) or (length > tote_mid and height > tote_mid) or (width > tote_mid or height > tote_mid):
        decision = False
    elif length > tote_max or width > tote_max or height > tote_max:
        decision = False
    elif unit_weight > 60: #pounds
        decision = False
    elif item_volume*4 > tote_vol: #will less than 4 items fit in a tote
        decision = False
    elif float(daily_forecast)*15*item_volume <= tote_vol*5: #will 14 DOS fit in 5 totes or fewer
        decision = False
    elif category == True: # Bed and Bedding product_category_level3 items are not AP eligible
        decision = False
    else:
        decision = True
    return decision
```

### Appendix: Supplier-Location Query

```{SQL}
select vendor_status, vendor_distribution_method, vl.*
from chewybi.vendor_location vl
where snapshot_date=current_date
        and vendor_number='XYZ'
        and location_code in (select location_code from chewybi.locations where fulfillment_active = true and product_company_description = 'Chewy' and location_warehouse_type = 0)
order by location_code
```

### Appendix: Item-Location Forecast Query

```{SQL}
select p.product_part_number
        , location_code
        , forecast_current_forecast_dt
        , forecast_current_manual_forecast_quantity
        , forecast_current_statistical_forecast_quantity
from chewybi.forecast_current f
join chewybi.products p on p.product_key = f.product_key
join chewybi.locations l on f.location_key=l.location_key
where forecast_current_date = current_date
        and forecast_current_forecast_dt between current_date and current_date + 180
--        and location_code='CLT1'
        and product_part_number = '123456'
order by 1,2,3;
```

### Appendix: Item Launch and Forecast Complete Dates' Query

```{SQL}
select product_part_number
      , product_abc_code
      , mercury_ovrrd_launch_dt
      , mercury_launch_dt
      , product_forecast_completed_flag
      , product_forecast_completed_dttm::date
      , product_published_dttm
      , product_published_flag
      ,*
from chewybi.product_lifecycle_snapshot
where snapshot_date=current_date
        and product_part_number='277453';
```

### Appendix: Override Disposition Script

In Mercury DB:

```{SQL}
-- Need zero forecast items
select distinct item_Id
from mercury.item_attr
where 1=1
        and attr_cd='ZERO_FORECAST_IND' and attr_val='true'
        and item_id not like 'RX%'
;
-- Replacement items
select item_id
        , relation_typ
        , related_item_id
from item_relation
where 1=1
        and relation_typ = 'RELATED_ITEM'
order by 1
;
```

In Chewybi DB:

```{SQL}
drop table if exists zero_forecast_items;
create local temp table zero_forecast_items
        ( item_id varchar(7))
on commit preserve rows;
copy zero_forecast_items
from local 'C:\users\cmorris10\Downloads\zeroforecast_items.csv'
parser fcsvparser(delimiter = ',');

drop table if exists related_items;
create local temp table related_items (
        item_id varchar(7)
        , relation varchar(20)
        , related_item_id varchar(7)
)
on commit preserve rows;
copy related_items
from local 'C:\users\cmorris10\Downloads\related_items.csv'
parser fcsvparser(delimiter = ',');

drop table if exists final_related_itemset;
create local temp table final_related_itemset on commit preserve rows as
        select r.item_id
                , p.product_abc_code
                , p.product_discontinued_flag
                , u.cartonization_flag
                , r.relation
                , r.related_item_id
                , rp.product_abc_code as related_abc_code
                , rp.product_discontinued_flag as related_disco_flag
                , ru.cartonization_flag as related_cartonization_flag
        from related_items r
        join chewybi.products p
                on r.item_id=p.product_part_number
        join chewybi.products rp
                on r.related_item_id=rp.product_part_number
        left join aad.t_item_uom u
                on u.item_number=r.item_id
                and u.wh_id='AVP1'
        left join aad.t_item_uom ru
                on ru.item_number=r.item_id
                and ru.wh_id='AVP1'
        where 1=1
                and p.product_discontinued_flag is false --need the item to be assorted to be non-disco
                and rp.product_discontinued_flag is true --related item is discontinued and thus the related item will not be assorted
        order by 1
;

-- These overrides should not be removed as their overrides are needed. These overrides come from the manual tracker that I have located at: L:\Cory Morris\Assortment Overrides\Override Tracking\Item Assortment Override Reason Code Tracker v1.xlsx

drop table if exists tracked_overrides_manual;
create local temp table tracked_overrides_manual
        (
          item_id varchar(7)
        , location_cd varchar(5)
        )
on commit preserve rows;
copy tracked_overrides_manual
from local 'C:\users\cmorris10\Downloads\overrides.csv'
parser fcsvparser(delimiter = ',');

drop table if exists tiered_racking_items;
create local temp table tiered_racking_items
        ( item_id varchar(7)
        , reason varchar(100)
        )
on commit preserve rows;
copy tiered_racking_items
from local 'L:\Cory Morris\Assortment Overrides\Alerts\Restricted_Tiered_racking_items.csv'
parser fcsvparser(delimiter = ',');

drop table if exists locations;
create local temp table locations on commit preserve rows as
        select location_code
        from chewybi.locations
        where fulfillment_active = true
                and product_company_description = 'Chewy'
                and location_warehouse_type = 0
;

-- Need to update this to use Emilio's logic
drop table if exists uom_items;
create local temp table uom_items on commit preserve rows as
        select item_number::varchar(7)
                , case when cartonization_flag='NO' then false else true end as cartonization_flag --To get it to match the assortment configs
        from aad.t_item_uom
        where cartonization_flag = 'NO'
                and wh_id='AVP1'
        order by item_number
;

drop table if exists freshpet_items;
create local temp table freshpet_items on commit preserve rows as
        select product_part_number
        from chewybi.products p
        where lower(p.product_purchase_brand) like 'freshpet%'
                and p.product_discontinued_flag is false
        order by product_part_number
;

drop table if exists assortment_overrides;
create local temp table assortment_overrides on commit preserve rows as
        select *
        from chewybi.mercury_assortment_override
        where location_cd in (select location_code from locations)
;

drop table if exists hazmat_items;
create local temp table hazmat_items on commit preserve rows as
        select im.item_number
                , im.wh_id
                , haz_material
                , hazmat_master_id
        from aad.t_item_master im
        left join aad.t_item_uom iu on im.item_number=iu.item_number
        where 1=1
                and hazmat_master_id in ('2', '3', '4', '5', '6', '7', '8', '9')
                and haz_material='Y'
                and im.wh_id='AVP1'
;

drop table if exists data_set;
create local temp table data_set on commit preserve rows as
        with data_set as (
                select  ao.item_id
                        , ao.location_cd
                        , ao.direction
--                        , ao.lifecycle_cd
--                        , case when haz_material='Y' then true else false end as HAZMAT_item
                        , p.product_abc_code
                        , uom.cartonization_flag--coalesce(uom.cartonization_flag,p.cartonization_flag) as cartonization_flag
        --                , hazmat_master_id
                        -- add product publish dates then calc tiem since publish- if longer than 90 days we should re-assort the zero-forecast items
                        , p.product_status
                        , p.product_discontinued_flag
                        , p.product_published_dttm::date
                        , p.product_first_publication_dttm::date
                        , current_date - product_published_dttm::date as days_since_published
                        , pa."attribute.fresh"
                        , pa."attribute.refrigerated"
                        , pa."storeitem.freezerrequired"
                        --Nedd to add Emilio's checker for UOM here
                        , greatest(p.product_warehouse_packaged_length, p.product_warehouse_packaged_width, p.product_warehouse_packaged_height) as largest_whse_dimension
                        , greatest(p.product_length, p.product_width, p.product_height) as largest_product_dimension
                        , case when tr.item_id is not null and ao.location_cd in ('AVP1','AVP2','CFC1','MCI1','MCO1','WFC2') then true else false end as cannot_assort_tiered_racking
                        , case when fi.product_part_number is NULL then false else true end as is_freshpet_item
                        , case when uom.cartonization_flag is false then true else false end as is_UOM_item
                        , case when p.product_dropship_flag is true then true else false end as is_dropship_item
                        , case when tom.item_id is null then false else true end as override_on_tracking_sheet
                        , case when z.item_id is not null then true else false end as is_zero_forecast_item
                        , case when (current_date - p.product_first_publication_dttm::date <= 180 or p.product_first_publication_dttm is NULL) and p.product_abc_code in ('D','E') and ao.location_cd in ('CFC1','WFC2') then true else false end as cfc_wfc_velocity_exclusion
                        , case when ri.item_id is not null then true else false end as is_related_item_discontinued
                --        , case when ao.location_cd in ('AVP2','MCI2','MCI1') and ao.direction='in'
                from assortment_overrides ao
                left join chewybi.products p
                        on ao.item_id=p.product_part_number
                left join chewybi.product_attributes pa
                        on p.product_key=pa."products.product_key"
                left join freshpet_items fi
                        on ao.item_id=fi.product_part_number
                left join uom_items uom
                        on ao.item_id=uom.item_number
                left join tracked_overrides_manual tom
                        on ao.item_id=tom.item_id
                        and ao.location_cd=tom.location_cd
                left join zero_forecast_items z
                        on ao.item_id=z.item_id
--                left join hazmat_items hz
--                        on ao.item_id=hz.item_number
--                        and ao.location_cd=hz.wh_id
                left join tiered_racking_items tr
                        on ao.item_id=tr.item_id
                left join final_related_itemset ri
                        on ao.item_id=ri.item_id
                where 1=1
--                        and ao.item_id='99577'
                order by ao.item_id, ao.direction, ao.location_cd
        )
        select d.*
                , ma.include
                , case  when product_status <> 'Active'/*lifecycle_cd <> 'active'*/ then 'REMOVE: Discontinued Item'
                        when is_dropship_item is true then 'REMOVE: Dropship Item'
                        when override_on_tracking_sheet is true then 'KEEP: on Override Tracker'
                        when is_related_item_discontinued is true and direction='in' and is_UOM_item is false and d.location_cd in ('AVP1','AVP2','EFC3','MCI1','RNO1') then 'KEEP: Related item is discontinued'
                        when is_related_item_discontinued is true and direction='in' and is_UOM_item is true and d.location_cd in ('AVP1','AVP2','EFC3','MCI1','RNO1') then 'REMOVE: UOM related item at non-UOM FC'
                        when is_related_item_discontinued is true and direction='out' and is_UOM_item is false and d.location_cd in ('AVP1','AVP2','EFC3','MCI1','RNO1') then 'UPDATE: change to Include'
                        when is_related_item_discontinued is true and direction='out' and is_UOM_item is true and d.location_cd in ('AVP1','AVP2','EFC3','MCI1','RNO1') then 'REMOVE: UOM related item and will not be re-assorted as item is disco'
--                        when d.location_cd in ('AVP2','MCI1') and direction='in' and coalesce(largest_whse_dimension,largest_product_dimension) >25 then 'REMOVE: Not Autostore eligible'
                        when d.location_cd in ('MCI2') then 'REMOVE: de-activated FC (MCI2)'
                        when d.location_cd in ('AVP2','MCI1') and product_abc_code = 'E' then 'KEEP: E-velocity item at Fixed Assortment FC'
                        when d.location_cd in ('MDT1') and direction='in' then 'KEEP: MDT1 singles assorted there'
                        when is_freshpet_item is true then 'KEEP: Freshpet item assorted to non-frozen FC'
                        when d.location_cd in ('AVP2','MCI1') and product_abc_code <> 'E' then 'REMOVE: ABCD item at AVP2+MCI1'
                        when "storeitem.freezerrequired" is true and d.location_cd not in ('CFF1','WFF2') then 'REMOVE: Frozen Item - assort to CFF + WFF only'
                        when is_UOM_item is true and direction='in' and d.location_cd in ('AVP1','AVP2','EFC3','MCI1') then 'REMOVE: non-UOM only FC'
                        when is_zero_forecast_item is true and d.location_cd not in ('AVP1','PHX1') and direction='out' and product_abc_code in ('A','B','C','D') then 'REMOVE: Zero Forecast item no longer applies'
                        when is_zero_forecast_item is true and d.location_cd not in ('AVP1','PHX1') and direction='out' and coalesce(product_abc_code,'E')='E' then 'KEEP: Zero Forecast item still applies'
                        when is_zero_forecast_item is true and d.location_cd in ('AVP1','PHX1') and direction='out' then 'REMOVE: Zero Forecast item not assorted AVP1+PHX1'
                        when cannot_assort_tiered_racking is true and direction='out' then 'KEEP: Tiered Racking Item exclusion'
                        when cannot_assort_tiered_racking is true and direction='in' then 'REMOVE: Tiered Racking Item inclusion'
                        when d.item_id in ('266248','266244') and d.location_cd='AVP2' and direction='out' then 'KEEP: oversized bed'
--                        when override_on_tracking_sheet is true then 'REMOVE: UOM exclusion not needed' --Only needed when overrides exist for UOM items being forced from assortment until next monthly run
                        -- Add in filter for items forced in when they don't meet building configs - these should be re-evaluated
                        when include='N'then 'REMOVE: item does NOT meet Assortment Configs'
                        when include='Y' and "storeitem.freezerrequired" is true then 'REMOVE: Frozen Item - assort to CFF + WFF only'
                        -- Add CFC/WFC
--                        when d.location_cd in ('CFC1','WFC2') and direction='out' and coalesce(product_abc_code,'E') in ('D','E') then 'KEEP: CFC/WFC D+E item'
                        when d.location_cd in ('CFC1','WFC2') then 'KEEP: CFC/WFC overrides - need to reevaluate'
                        else 'REMOVE'
                        end as Action
                , case when d.location_cd in ('AVP2','MCI1') --New FCs with Fixed assortments
                                and "storeitem.freezerrequired" is not true --non-freezer items
                                and is_UOM_item is false --non-UOM only
                                and coalesce(largest_whse_dimension,largest_product_dimension) <=25 --Autostore requirement
                                and product_abc_code is not null
                                and product_discontinued_flag is false
                        then 'Y'
                        else 'N'
                        end as subaction
        from data_set d
        left join chewybi.mercury_assortment_config ma
                on d.location_cd=ma.location_cd
                and d.product_abc_code=ma.abc_cd
                and d.is_uom_item=ma.uom_ind
        order by item_id, location_cd
        segmented by hash(item_id, location_cd) all nodes
;

---- Create the below table when you want to get impact on Assortment post-changes
drop table if exists overrides;
create local temp table overrides on commit preserve rows as
with removals as (
        select item_id
                , location_cd
                , direction
                , action
                , subaction
--                , lifecycle_cd
--                , hazmat_item
                , product_abc_code
                , cartonization_flag
                , product_status
                , product_discontinued_flag
                , "attribute.fresh"
                , "attribute.refrigerated"
                , "storeitem.freezerrequired"
                , largest_whse_dimension
                , largest_product_dimension
                , cannot_assort_tiered_racking
                , is_freshpet_item
                , is_UOM_item
                , is_dropship_item
                , case  when (include='Y' and "storeitem.freezerrequired" is true) then 'N'
                        when product_status <> 'Active' then 'N'--lifecycle_cd <> 'active' then 'N'
--                        when product_discontinued_flag is true then 'N'
                        when product_abc_code is null then 'N'
                        when is_dropship_item is true then 'N'
                        else include
                  end as meets_configuration
        from data_set
        where 1=1
--                and Action like 'REMOVE%'
--                and location_cd  in ()
        order by 1,2
)
select * from removals;

drop table if exists sandbox_supply_chain.cmorris10_override_removal;
create table sandbox_supply_chain.cmorris10_override_removal as
        select current_date as snapshot_date, * from overrides
;
```

### Appendix: Eligible Vendor for Item-Location Query

```{SQL}
select cil.location
        , cil.item
        , cil.supplier
        , v.vendor_distribution_method
        , cil.SRCLOCATION
        , cil.avedemqty::numeric(12,6)
        , MINRESLOT
        , case when supplier in (select distinct location_code from chewybi.locations where fulfillment_active = true and product_company_description = 'Chewy' and location_warehouse_type = 0)
                        then true
                when supplier = '' then true
                when supplier = '?' then true
                else false
                end as not_a_valid_source
from chewy_prod_740.C_ITEMLOCATION cil
left join chewybi.vendors v on split_part(cil.SUPPLIER, '-', 1)=v.vendor_number
where cil.item = '277453'
order by cil.item, cil.location
```

### Appendix: Replacment Items Query

The `related_item_id` is the item whose velocity code is used to assort the `item_id` SKU.

```{SQL}
select *
from item_relation
where relation_typ = 'RELATED_ITEM'
        and item_id = '160154';
```

### Appendix: De-bundle Item Query

```{SQL}
Select ITEMKIT as child_sku
        , ITEMCOMP as parent_sku
from chewy_prod_740.C_KITCOMP deb
where ITEMKIT = '184968' or ITEMCOMP='184968'
;
```

### Appendix: Current Assortment Query

```{SQL}
select i.location_code
        , p.product_abc_code
        , COUNT(*) as assorted_item_count
        , SUM(case when coalesce(i.inventory_snapshot_on_hand_quantity,0) <= 0 then 1 else 0 end) as "Items w/ Zero units OH"
from chewybi.inventory_snapshot i
join chewybi.products p using(product_part_number)
where 1=1
        and i.inventory_snapshot_snapshot_dt = current_date
        and i.inventory_snapshot_managed_flag is true
        and i.location_code in (select location_code from chewybi.locations where fulfillment_active = true and product_company_description = 'Chewy' and location_warehouse_type in (0,1)) -- all retail and freezer FCs
        and coalesce(p.product_discontinued_flag,false)=false
        and p.product_dropship_flag is false
        and p.product_abc_code is not null
group by 1,2
order by 1,2;
```

### Appendix: New FC Launch Script

The Github repository for this script is located [here](https://github.com/cmorris10-chwy/Automation/blob/pr/bokeh_dash/New_FC_Launch/run/generate_new_buidling_assortment.py).