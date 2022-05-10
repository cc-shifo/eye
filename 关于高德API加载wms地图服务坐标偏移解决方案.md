前段时间花了半天的时间解决了这个问题，一直想记录下来，就是说过撂过，刚看完《欢乐颂》，让我思考了不少。乘着现在还有思维，下面说说解决办法。

难点： 1、geoserver发布的wms服务需要知道两个参数（图层名称和地图范围）
  eg："http://www.wochmoc.org.cn/geoserver/wms?LAYERS="+图层名称+"&FORMAT=image%2Fpng&TRANSPARENT=TRUE&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A900913&BBOX="+地图范围+“&WIDTH=256&HEIGHT=256”；

            2、高德api传出来的参数是行列号和等级

问题就来了，如何将高德的行列号转换成正确的地图范围就成了解决问题的关键。

思路：
      （1）在墨卡托投影系中先根据行列号求出瓦片的范围（米）；
       （2）将瓦片范围转换为度；
       （3）利用高德api转换工具求出正常坐标和高德坐标的差；
        （4）在（2）的坐标加上差值得到正确坐标，即可获取到正确的瓦片。 

注：坐标转换计算方法
http://www.maptiler.org/google-maps-coordinates-tile-bounds-projection/

最后的结果如下：



关键代码如下：



    TileOverlay scopeTileOverlay;
    int titleSize=256;
    double initialResolution=156543.03392804062;//2*Math.PI*6378137/titleSize;//
    double  originShift=20037508.342789244;//2*Math.PI*6378137/2.0;// 

    /**
     * 根据像素、等级算出坐标
     * @param p
     * @param zoom
     * @return
     */
    private double  Pixels2Meters(int p,int zoom){
        return p*Resolution(zoom)-originShift;
    }
    /**
     * 根据瓦片的x/y等级返回瓦片范围
     * @param tx
     * @param ty
     * @param zoom
     * @return
     */
    private String  TitleBounds(int tx,int ty,int zoom){
        double minX=Pixels2Meters(tx*titleSize,zoom);
        double maxY=-Pixels2Meters(ty*titleSize,zoom);
        double maxX=Pixels2Meters((tx+1)*titleSize,zoom);
        double minY=-Pixels2Meters((ty+1)*titleSize,zoom);

        //转换成经纬度
        minX=Meters2Lon(minX);
        minY=Meters2Lat(minY);
        maxX=Meters2Lon(maxX);
        maxY=Meters2Lat(maxY);
        //经纬度转换米
        //        minX=Lon2Meter(minX);
        //        minY=Lat2Meter(minY);
        //        maxX=Lon2Meter(maxX);
        //        maxY=Lat2Meter(maxY);
        CoordinateConverter converter  = new CoordinateConverter();
        converter.from(CoordType.GPS);
        converter.coord(new LatLng( minY,minX));
        LatLng min=converter.convert();
        converter.coord(new LatLng(maxY,maxX));
        LatLng max=converter.convert();
        minX=Lon2Meter(-min.longitude+2*minX);
        minY=Lat2Meter(-min.latitude+2*minY);
        maxX=Lon2Meter(-max.longitude+2*maxX);
        maxY=Lat2Meter(-max.latitude+2*maxY);
        return Double.toString(minX)+","+Double.toString(minY)+","+Double.toString(maxX)+","+Double.toString(maxY)+"&WIDTH=256&HEIGHT=256";
    }

    /**
     * 计算分辨率
     * @param zoom
     * @return
     */
    private double Resolution(int zoom){
        return initialResolution/(Math.pow(2, zoom));
    }

    /**
     * X米转经纬度
     */
    private double Meters2Lon(double mx){
        double lon=(mx/originShift)*180.0;
        return lon;
    }
    /**
     * Y米转经纬度
     */
    private double Meters2Lat(double my){
        double lat=(my/originShift)*180.0;
        lat=180.0/Math.PI*(2*Math.atan(Math.exp(lat*Math.PI/180.0))-Math.PI/2.0);
        return lat;
    }
    /**
     * X经纬度转米
     */
    private double Lon2Meter(double lon){
        double mx= lon*originShift/180.0;
        return mx;
    }
    /**
     * Y经纬度转米
     */
    private double Lat2Meter(double lat){
        double my=Math.log(Math.tan((90+lat)*Math.PI/360.0))/(Math.PI/180.0
                );
        my=my*originShift/180.0;
        return my;
    }

    String url ="";

    /**
     * 添加wms图层
     */
    protected void addScope(){
        Vector<ScopeModel> models;
        if(MyApplication.heritageCode.equals("11035")||MyApplication.heritageCode.contains("11035")){
            url = "http://110.72.251.22:8080/geoserver/hsyh/wms?LAYERS=hsyh:hsyh_ycq_l,hsyh:hsyh_ycqhcq_l&FORMAT=image%2Fpng&TRANSPARENT=TRUE&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A900913&BBOX=";
            models=    ScopeModel.quearyByCode("11035");
            if(models.size()==0) return;
        }else{
            models=    ScopeModel.quearyByCode(MyApplication.heritageCode);
            if(models.size()==0) return;
            url = "http://www.wochmoc.org.cn/geoserver/wms?LAYERS="+models.get(0).getLayer()+"&FORMAT=image%2Fpng&TRANSPARENT=TRUE&SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&STYLES=&SRS=EPSG%3A900913&BBOX=";
        }

        TileProvider tileProvider = new UrlTileProvider(256, 256) {
            @Override
            public URL getTileUrl(int x, int y, int zoom) {
                // TODO Auto-generated method stub
                try {
                    return new URL(url+ TitleBounds(x,y,zoom));
                } catch (MalformedURLException e) {
                    e.printStackTrace();
                }
                return null;
            }
        };
        if (tileProvider != null) {
            scopeTileOverlay = aMap.addTileOverlay(new TileOverlayOptions()
            .tileProvider(tileProvider)
            .diskCacheDir("/storage/amap/cache").diskCacheEnabled(true)
            .diskCacheSize(100));
        }
    }

