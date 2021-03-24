
`TiTiler` is designed to help user customize input/output for each endpoint. This section goes over some simple customization examples.

### Custom DatasetPathParams for `path_dependency`

One common customization could be to create your own `path_dependency` (used in all endpoints).

Here an example which allow a mosaic to be passed by a `mosaic` name instead of a full S3 url.

```python
import os
import re
from titiler.dependencies import DefaultDependency
from typing import Optional, Type
from rio_tiler.io import BaseReader
from fastapi import HTTPException, Query

MOSAIC_BACKEND = os.getenv("TITILER_MOSAIC_BACKEND")
MOSAIC_HOST = os.getenv("TITILER_MOSAIC_HOST")


def MosaicPathParams(
    mosaic: str = Query(..., description="mosaic name")
) -> str:
    """Create dataset path from args"""
    # mosaic name should be in form of `{user}.{layername}`
    if not re.match(self.mosaic, r"^[a-zA-Z0-9-_]{1,32}\.[a-zA-Z0-9-_]{1,32}$"):
        raise HTTPException(
            status_code=400,
            detail=f"Invalid mosaic name {self.mosaic}.",
        )

    return f"{MOSAIC_BACKEND}{MOSAIC_HOST}/{self.mosaic}.json.gz"
```

The endpoint url will now look like: `{endoint}/mosaic/tilejson.json?mosaic=vincent.mosaic`


### Custom TMS

```python
from morecantile import tms, TileMatrixSet
from rasterio.crs import CRS

from titiler.endpoint.factory import TilerFactory

# 1. Create Custom TMS
EPSG6933 = TileMatrixSet.custom(
    (-17357881.81713629, -7324184.56362408, 17357881.81713629, 7324184.56362408),
    CRS.from_epsg(6933),
    identifier="EPSG6933",
    matrix_scale=[1, 1],
)

# 2. Register TMS
tms = tms.register([EPSG6933])

# 3. Create ENUM with available TMS
TileMatrixSetNames = Enum(  # type: ignore
    "TileMatrixSetNames", [(a, a) for a in sorted(tms.list())]
)

# 4. Create Custom TMS dependency
def TMSParams(
    TileMatrixSetId: TileMatrixSetNames = Query(
        TileMatrixSetNames.WebMercatorQuad,  # type: ignore
        description="TileMatrixSet Name (default: 'WebMercatorQuad')",
    )
) -> TileMatrixSet:
    """TileMatrixSet Dependency."""
    return tms.get(TileMatrixSetId.name)

# 5. Create Tiler
COGTilerWithCustomTMS = TilerFactory(
    reader=COGReader,
    tms_dependency=TMSParams,
)
```

### Add a MosaicJSON creation endpoint
```python

from dataclasses import dataclass
from typing import List, Optional

from titiler.endpoints.factory import MosaicTilerFactory
from titiler.errors import BadRequestError
from cogeo_mosaic.mosaic import MosaicJSON
from cogeo_mosaic.utils import get_footprints
import rasterio

from pydantic import BaseModel


# Models from POST/PUT Body
class CreateMosaicJSON(BaseModel):
    """Request body for MosaicJSON creation"""

    files: List[str]              # Files to add to the mosaic
    url: str                      # path where to save the mosaicJSON
    minzoom: Optional[int] = None
    maxzoom: Optional[int] = None
    max_threads: int = 20
    overwrite: bool = False


class UpdateMosaicJSON(BaseModel):
    """Request body for updating an existing MosaicJSON"""

    files: List[str]              # Files to add to the mosaic
    url: str                      # path where to save the mosaicJSON
    max_threads: int = 20
    add_first: bool = True


@dataclass
class CustomMosaicFactory(MosaicTilerFactory):

    def register_routes(self):
        """Update the class method to add create/update"""
        super().register_routes()
        # new methods/endpoint
        self.create()
        self.update()

    def create(self):
        """Register / (POST) Create endpoint."""

        @self.router.post(
            "", response_model=MosaicJSON, response_model_exclude_none=True
        )
        def create(body: CreateMosaicJSON):
            """Create a MosaicJSON"""
            # Write can write to either a local path, a S3 path...
            # See https://developmentseed.org/cogeo-mosaic/advanced/backends/ for the list of supported backends

            # Create a MosaicJSON file from a list of URL
            mosaic = MosaicJSON.from_urls(
                body.files,
                minzoom=body.minzoom,
                maxzoom=body.maxzoom,
                max_threads=body.max_threads,
            )

            # Write the MosaicJSON using a cogeo-mosaic backend
            src_path = self.path_dependency(body.url)
            with rasterio.Env(**self.gdal_config):
                with self.reader(
                    src_path, mosaic_def=mosaic, reader=self.dataset_reader
                ) as mosaic:
                    try:
                        mosaic.write(overwrite=body.overwrite)
                    except NotImplementedError:
                        raise BadRequestError(
                            f"{mosaic.__class__.__name__} does not support write operations"
                        )
                    return mosaic.mosaic_def

    def update(self):
        """Register / (PUST) Update endpoint."""

        @self.router.put(
            "", response_model=MosaicJSON, response_model_exclude_none=True
        )
        def update_mosaicjson(body: UpdateMosaicJSON):
            """Update an existing MosaicJSON"""
            src_path = self.path_dependency(body.url)
            with rasterio.Env(**self.gdal_config):
                with self.reader(src_path, reader=self.dataset_reader) as mosaic:
                    features = get_footprints(body.files, max_threads=body.max_threads)
                    try:
                        mosaic.update(features, add_first=body.add_first, quiet=True)
                    except NotImplementedError:
                        raise BadRequestError(
                            f"{mosaic.__class__.__name__} does not support update operations"
                        )
                    return mosaic.mosaic_def

```