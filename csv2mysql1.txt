'use strict';
var bodyParser = require('body-parser');
const parse      = require('csv-parse');
const util       = require('util');
const fs         = require('fs');
const path       = require('path');
const mysql      = require('mysql');
const async      = require('async');
const co         = require('co');
const csvHeaders = require('csv-headers');
const leftpad    = require('leftpad');
const dbhost =  '127.0.0.1'; 			// process.argv[2]; //127.0.0.1
const dbuser =  'root'; 				// process.argv[3]; //root
const dbpass = 	'root'; 				//process.argv[4]; //root
const dbname =  'sample'; 			//process.argv[5]; //sample
const tblnm  =  'apidetails'; 			//process.argv[6]; // apidetails
//const csvfn  = 	'./apidetails.csv'					//process.argv[7]; 	//apidetails.csv

var express = require('express');
var app = express();

//welcome page
app.get('/',function(req,res){
      res.sendFile(__dirname + "/upload.html");
});

//upload request
app.use(bodyParser.urlencoded({ extended: true })); 

app.post('/upload',function(req,res){
	console.log('inside post');
	const csvfn ='./'+req.body.fileToUpload;
	console.log('filename'+csvfn);
	
    /*upload(req,res,function(err) {
        if(err) {
            return res.end("Error uploading file.");
        }
		const csvfn ='./'+req.body.upload;
		
		console.log('filename'+csvfn);
		upload(csvfn)
        res.end("File is uploaded");
    });*/
	 
		//const csvfn ='./'+req.body.upload;
		
		console.log('filename'+csvfn);
		upload(csvfn);
		//var html ="<html><body>upload successful </br><form action="/upload" method="post"><select></select> <input type="submit" value="Upload csv file" name="submit"></form></body></html>" 
      var html =  "<!DOCTYPE html>\n<html>\n    <head>\n    </head>\n   <body>\n     <center> <h1>File Upload successful!!</h1>\n<form><select><option>select</option></select>  </fotm></center> </body>\n</html>";

		res.send(html);
});

app.listen(3000,function(){
    console.log("Working on port 3000");
	
});
function upload(file){
	console.log('inside upload'+file);
new Promise((resolve, reject) => {
    csvHeaders({
        file      : file,
        delimiter : ','
    }, function(err, headers) {
        if (err) reject(err);
        else resolve({ headers });
    });
})
.then(context => {
    return new Promise((resolve, reject) => {
        context.db = mysql.createConnection({
            host     : dbhost,
            user     : dbuser,
            password : dbpass,
            database : dbname
        });
        context.db.connect((err) => {
            if (err) {
                console.error('error connecting: ' + err.stack);
                reject(err);
            } else {
                resolve(context);
            }
        });
    })
})
.then(context => {
    return new Promise((resolve, reject) => {
        context.db.query(`DROP TABLE IF EXISTS ${tblnm}`,
        [ ],
        err => {
            if (err) reject(err);
            else resolve(context);
        })
    });
})
.then(context => {
    return new Promise((resolve, reject) => {
        var fields = '';
        var fieldnms = '';
        var qs = '';
        context.headers.forEach(hdr => {
            hdr = hdr.replace(' ', '_');
            if (fields !== '') fields += ',';
            if (fieldnms !== '') fieldnms += ','
            if (qs !== '') qs += ',';
            fields += ` ${hdr} TEXT`;
            fieldnms += ` ${hdr}`;
            qs += ' ?';
        });
        context.qs = qs;
        context.fieldnms = fieldnms;
        // console.log(`about to create CREATE TABLE IF NOT EXISTS ${tblnm} ( ${fields} )`);
        context.db.query(`CREATE TABLE IF NOT EXISTS ${tblnm} ( ${fields} )`,
        [ ],
        err => {
            if (err) reject(err);
            else resolve(context);
        })
    });
})
.then(context => {
    return new Promise((resolve, reject) => {
        fs.createReadStream(file).pipe(parse({
            delimiter: ',',
            columns: true,
            relax_column_count: true
        }, (err, data) => {
            if (err) return reject(err);
            async.eachSeries(data, (datum, next) => {
                 console.log(`about to run INSERT INTO ${tblnm} ( ${context.fieldnms} ) VALUES ( ${context.qs} )`);
                var d = [];
                try {
                    context.headers.forEach(hdr => {
                        d.push(datum[hdr]);
                    });
                } catch (e) {
                    console.error(e.stack);
                }
                 console.log(`${d.length}: ${util.inspect(d)}`);
                if (d.length > 0) {
                    context.db.query(`INSERT INTO ${tblnm} ( ${context.fieldnms} ) VALUES ( ${context.qs} )`, d,
                    err => {
                        if (err) { console.error(err); next(err); }
                        else setTimeout(() => { next(); });
                    });
                } else { console.log(`empty row ${util.inspect(datum)} ${util.inspect(d)}`); next(); }
            },
            err => {
                if (err) reject(err);
                else resolve(context);
            });
        }));
    });
})
.then(context => { context.db.end(); })
.catch(err => { console.error(err.stack); });

}