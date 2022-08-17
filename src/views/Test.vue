<template>
    <div>
        <form v-on:submit="submit">
            User company address: <br>
            <input id="company_address" name="company_address" v-model="company_address">
            <br>
            User address (input type: user|address): <br>
            <textarea id="text" name="text" v-model="text" rows="8" cols="50">
    </textarea>
            <br>
            <button type="submit">Submit</button>
        </form>

        <div>
            User company address: <br>
            <textarea id="text" rows="8" cols="50">{{ this.company_address_res }}</textarea>
            <br>
            User location: <br>
            <textarea id="text" rows="8" cols="50">{{ this.outputdata }}</textarea>
            <br>
            User can't query: <br>
            <textarea id="text" rows="8" cols="50">{{ this.outputdatacantquery }}</textarea>
        </div>

        <div>
            User distance to company site:
            <table style="width: 100%;">
                <thead>
                    <th>User</th>
                    <th>Address</th>
                    <th>Distance</th>
                    <th>OPs</th>
                </thead>
                <tbody v-for="child in usersInfo">
                    <td>{{ child.name }}</td>
                    <td>{{ child.address }}</td>
                    <td v-if="child.is_query_complete">{{ calculateLong(child.longlat.latitude,
                            child.longlat.longtitude,
                            company_address_res.latitude, company_address_res.longtitude)
                    }}</td>
                    <td v-else>Cannot query user</td>
                    <td v-if="child.is_query_complete"><button @click="centerMap(child.longlat.longtitude, child.longlat.latitude)">Center</button></td>
                </tbody>
            </table>
        </div>

        <div ref="map-root" style="width: 100%; height: 40vw">
        </div>
    </div>
</template>

<script>
// @ is an alias to /src

import axios from 'axios'
import View from 'ol/View'
import Map from 'ol/Map'
import TileLayer from 'ol/layer/Tile'
import OSM from 'ol/source/OSM'

import { fromLonLat } from 'ol/proj';
import { Icon, Style, Text, Fill, Stroke } from 'ol/style';


import VectorLayer from 'ol/layer/Vector';
import VectorSource from 'ol/source/Vector'
import Point from 'ol/geom/Point';
import Feature from 'ol/Feature';

import {transform} from 'ol/proj';


// importing the OpenLayers stylesheet is required for having
// good looking buttons!
import 'ol/ol.css'

export default {
    data() {
        return {
            average: 0,
            usersInfo: [],
            outputdata: [],
            outputdatacantquery: [],
            company_address_res: { longtitude: "", latitude: "" }
        }
    },
    methods: {
        calculateLong(lat1, lon1, lat2, lon2) {
            const R = 6371000; // metres
            const φ1 = lat1 * Math.PI / 180; // φ, λ in radians
            const φ2 = lat2 * Math.PI / 180;
            const Δφ = (lat2 - lat1) * Math.PI / 180;
            const Δλ = (lon2 - lon1) * Math.PI / 180;

            const a = Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
                Math.cos(φ1) * Math.cos(φ2) *
                Math.sin(Δλ / 2) * Math.sin(Δλ / 2);
            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

            const d = R * c; // in metres
            return d
        },
        async submit(e) {
            e.preventDefault()
            console.log(this.company_address)
            // company address query
            let url = `https://nominatim.openstreetmap.org/search?q=${this.company_address}&format=json&polygon=1&addressdetails=1`

            let data = await axios.get(url.replaceAll(" ", "+"))

            try {
                this.company_address_res.longtitude = data.data[0].lon
                this.company_address_res.latitude = data.data[0].lat
            }
            catch (e) {
                this.company_address_res = "Cannot query company address"
            }

            let myArray = this.text.split("\n");
            console.log(myArray)
            let users_cant_query = []
            let users = []
            for (let i = 0; i < myArray.length; i++) {
                let user = { name: "", address: "", is_query_complete: false, longlat: { longtitude: "", latitude: "" } }
                var splitAddress = myArray[i].split("|")
                let url = `https://nominatim.openstreetmap.org/search?q=${splitAddress[1]}&format=json&polygon=1&addressdetails=1`

                let data = await axios.get(url.replaceAll(" ", "+"))

                user.name = splitAddress[0]
                user.address = splitAddress[1]
                try {

                    user.longlat.longtitude = data.data[0].lon
                    user.longlat.latitude = data.data[0].lat

                    user.is_query_complete = true

                    users.push(user)
                }
                catch (e) {
                    users_cant_query.push(user)
                }

                this.usersInfo.push(user)
            }
            this.outputdatacantquery = users_cant_query
            this.outputdata = users
            console.log(this.usersInfo)

            this.drawMap(this.company_address_res, this.outputdata)
        },

        drawMap(company_address_res, outputdata) {
            console.log([company_address_res.latitude, company_address_res.longtitude])
            this.map = new Map({
                // the map will be created using the 'map-root' ref
                target: this.$refs['map-root'],
                layers: [
                    // adding a background tiled layer
                    new TileLayer({
                        source: new OSM() // tiles are served by OpenStreetMap
                    }),
                ],

                // the map view will initially show the whole world
                view: new View({
                    zoom: 16,
                    center: fromLonLat([company_address_res.longtitude, company_address_res.latitude])
                }),
            })

            // var layer = new VectorLayer({"test"
            //             new Feature({
            //                 geometry: new Point(fromLonLat([company_address_res.longtitude, company_address_res.latitude])),
            //                 text: "test"
            //             })
            //         ]
            //     })
            // });

            const iconStyle = new Style({
                image: new Icon({
                    anchorXUnits: 'fraction',
                    anchorYUnits: 'pixels',
                    src: '/marker.png',
                }),
            });

            const iconFeature = new Feature({
                geometry: new Point(fromLonLat([company_address_res.longtitude, company_address_res.latitude])),
                name: 'Company',
            });

            iconFeature.setStyle(iconStyle);


            var layer = new VectorLayer({
                source: new VectorSource({
                    features: [
                        iconFeature
                    ]
                })
            });

            this.map.addLayer(layer);


            // var layer = new VectorLayer({
            //     source: new VectorSource({
            //         features: [
            //             new Feature({
            //                 geometry: new Point(fromLonLat([company_address_res.longtitude, company_address_res.latitude])),
            //                 text: "test"
            //             })
            //         ]
            //     })
            // });

            let featuresVector = []
            console.log("test")

            for (let i = 0; i < outputdata.length; i++) {
                console.log([outputdata[i].longlat.longtitude, outputdata[i].longlat.latitude])
                


                let data = new Feature({
                    geometry: new Point(fromLonLat([outputdata[i].longlat.longtitude, outputdata[i].longlat.latitude])),
                    name: outputdata[i].name
                })

                // data.setStyle(iconStyle2);

                featuresVector.push(data)
            }

            console.log(featuresVector)

            let iconStyle2 = new Style({
                    image: new Icon({
                        anchorXUnits: 'fraction',
                        anchorYUnits: 'pixels',
                        src: '/mark2.png',
                    }),
                });

                let labelStyle2 = new Style({
                    text: new Text({
                        font: '12px Calibri,sans-serif',
                        overflow: true,
                        fill: new Fill({
                            color: '#000'
                        }),
                        stroke: new Stroke({
                            color: '#fff',
                            width: 3
                        })
                    })
                });

                let style = [iconStyle2, labelStyle2];


            var vector = new VectorLayer({
                source: new VectorSource({
                    features: featuresVector
                }),
                style: function(feature) {
                    labelStyle2.getText().setText(feature.get('name'));
                    return style;
                }
            });

            this.map.addLayer(vector);

            // this.map.view.setCenter(new Point(fromLonLat([0, 0])).transform('EPSG:4326', this.map.getView().getProjection()) )
            // console(this.map.view.getCenter())
            // this.map.getView().setCenter(transform([0,0], 'EPSG:4326', 'EPSG:3857'));
        },

        centerMap(longtitude, latitude) {
            // console.log(longtitude, latitude)
            // this.map.setView(new View({
            //     center: new Point(fromLonLat([105.79509316583201, 21.0342177])),
            //     zoom: 5
            // }));
            // this.map.setCenter(new Point(fromLonLat([105.79509316583201, 21.0342177])))

            // this.map.setCenter(new OpenLayers.LonLat(105.79509316583201, 21.0342177).transform('EPSG:4326', 'EPSG:3857'), zoom);

            this.map.getView().setCenter(transform([longtitude, latitude], 'EPSG:4326', 'EPSG:3857'));
            this.map.getView().setZoom(20)
        }
    }
}
</script>
