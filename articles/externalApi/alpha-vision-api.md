# Alpha-Vision API Integration

This document will summarize the steps for embedding Alpha Visions's interactive lot maps into an Angular application.

## Steps

1. Add the builder GUID to the environment files.  Since this is a read-only integration, and Alpha Vision only has production data, each environment will use the same builder GUID.

2. Add the Alpha Vision javascript to `index.html`

```js
<script src="https://apps.alpha-vision.com/api/siteplanAPIv1.4.js?v=1.0.42"></script>
```

3. Declare a variable to represent Alpha Vision within a component's TypeScript file

```js
declare let AVSiteplan: any;
```

4. Create a \<div> tag within a component's HTML file that will contain the interactive lot map.

5. Create an `ElementRef` to the \<div>

```js
@ViewChild('divSitePlan', { static: false }) divSitePlan: ElementRef;
```

6. Setup Alpha Vision specific models

```js
export interface AlphaVisionGeneralResponse {
  name: string;
  datatype: AlphaVisionDataType;
  eventtype: AlphaVisionEventType;
  data: AlphaVisionGeneralResponseData;
  siteplans?: AlphaVisionSitePlan[];
}

export interface AlphaVisionClickResponse {
  name: string;
  datatype: AlphaVisionDataType;
  eventtype: AlphaVisionEventType;
  data: AlphaVisionClickResponseData;
}

export interface AlphaVisionSelectMapResponse {
  data: AlphaVisionSitePlan;
  datatype: AlphaVisionDataType;
  eventtype: AlphaVisionEventType;
}

export interface AlphaVisionFilterLotsResponse {
  data: AlphaVisionFilterLotsResponseData;
  datatype: AlphaVisionDataType;
  eventtype: AlphaVisionEventType;
}

export interface AlphaVisionSelectLotResponse {
  data: AlphaVisionLot;
  datatype: AlphaVisionDataType;
  eventtype: AlphaVisionEventType;
}

export interface AlphaVisionPlan {
  id: string;
  name: string;
  number: string;
}

export interface AlphaVisionLot {
  lot: string;
  lotGroupName: string;
  lotImages: AlphaVisionImage[];
  lotLabel: string;
  lotNumber: string;
  plans: AlphaVisionPlan[];
  status: string;
}

export interface AlphaVisionImage {
  name: string;
  url: string;
}

export interface AlphaVisionAmenity {
  zoneName: string;
  medias: AlphaVisionImage[];
}

export class AlphaVisionSitePlan {
  amenities: AlphaVisionAmenity[];
  availablePlans: AlphaVisionPlan[];
  availableStatus: string[];
  lots: AlphaVisionLot[];
  mapName: string;
  siteplans: AlphaVisionSitePlan[];
}

export class AlphaVisionLotCounts {
  HomesiteLotsCount: number;
  AvailableHomesiteLotsCount: number;
  QmiLotsCount: number;
}

export enum AlphaVisionDataType {
  MasterMap = 'mastermap',
  SitePlan = 'siteplan',
  Error = 'error',
  Lot = 'lot',
  Amenity = 'amenity',
  Plan = 'plan',
  ZoomLevel = 'zoom_level',
  SVG = 'svg'
}

export enum AlphaVisionEventType {
  Init = 'init',
  Click = 'click',
  ShowLots = 'showLots()',
  ShowPlans = 'showPlans()',
  FilterLots = 'filterLots()',
  SelectLot = 'selectLot()',
  SelectMap = 'selectMap()',
  Reset = 'reset()',
  ExportStaticSVG = 'exportStaticSVG()'
}

export enum AlphaVisionLotStatusType {
  Available = 'Available',
  Model = 'Model',
  QuickMoveIn = 'Quick Move In',
  Sold = 'Sold',
  Unreleased = 'Unreleased'
}

export interface AlphaVisionGeneralResponseData extends AlphaVisionSitePlan, AlphaVisionLot, AlphaVisionPlan { }

export interface AlphaVisionClickResponseData extends AlphaVisionLot, AlphaVisionAmenity, AlphaVisionSitePlan { }

export interface AlphaVisionFilterLotsResponseData extends AlphaVisionLot, AlphaVisionSitePlan { }

```

6. Create the lot map SVG by instantiating a new Alpha Vision object with the following properties, usually supplying just a single status of `AlphaVisionLotStatusType.Available` for the array `mapInitStatuses`:

```js
this.mapAPI = new AVSiteplan(environment.alphaVisionBuilderGuid, // Builder GUID
    this.webCommunity.Id,                   // Web Community ID
    this.divSitePlan.nativeElement,         // Parent DOM Element
    res => this.mapGeneralCallback(res),    // Generic Map Callback
    res => this.mapClickEventCallback(res), // Event Map Callback (lot click/ amenity click)
    mapInitStatuses,                        // Status
    mapInitPlanIds);                        // Plans
```

7. Write the generic and event callback functions (simplified example below)

```js
private mapClickEventCallback(res: AlphaVisionClickResponse) {
    if (res.eventtype === AlphaVisionEventType.Click && res.datatype && res.data) {
      // Click callback can return 3 different types of click data responses,
      // (Lot, Amenity, or Map), depending on map element clicked.
      switch (res.datatype) {
        case AlphaVisionDataType.Lot:
          this.selectedLot = res.data as AlphaVisionLot;
          this.selectedLotId = this.selectedLot.lot;
          …
          break;
        default:
          break;
      }
    }
  }
```