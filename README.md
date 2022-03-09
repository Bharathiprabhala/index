const { MongoClient } = require("mongodb");
const mongodb = require("mongodb");
var express = require("express");
const bodyParser = require("body-parser");
const multer = require("multer");
const path = require("path");
require('dotenv').config()
var app = express();
app.use(bodyParser.json());
let db;
app.use(express.static(__dirname));
var storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, path.join(__dirname + "/uploads/"));
  },
  filename: function (req, file, cb) {
    cb(null, file.originalname + "-" + Date.now());
  },
});

MongoClient.connect(
  process.env.mongodb_url,
  { useNewUrlParser: true, useUnifiedTopology: true },
  function (err, client) {
    db = client.db();
    app.listen(process.env.port);
  }
);

const upload = multer({ dest: "uploads/" });
app.get("/api/v3/app/events/", (req, res, next) => {
  const id=  req.query.id;
  const { type, limit, page } = req.query;
 if(id){
  db.collection('events').findOne({_id: new mongodb.ObjectId(id)}, function(err, result) {
    if (err) throw err;
    res.send(result);
  }
  );
}  else if (type  && limit && page) {
  console.log(type, limit, page);
  const skip =  (page - 1) * limit;
  const query = {};
  if (type) {
    query.type = type;
  }
  db.collection("events")
    .find(query)
    .skip(skip)
    .limit(parseInt(limit))
    .toArray(function (err, result) {
      if (err) throw err;
      res.send(result);
    });
} else {
  db.collection('events').find().toArray(function(err, result) {
    if (err) throw err;
    res.send(result);
  }
  );
}
 });
app.put("/api/v3/app/events/:id",upload.array("files", 10), (req, res, next) => {
  const id = req.params.id;
  const event = req.body;
  const data = {
    ...event,
    files: req.files
  }
  db.collection("events").updateOne(
    { _id: new mongodb.ObjectId(id) },
 {
   $set: data
 },
    function (err, result) {
      if (err) throw err;
      res.send(result);
    }
  );
});
app.delete("/api/v3/app/events/:id", (req, res, next) => {
  let id = req.params.id;
  db.collection("events").deleteOne(
    { _id:  mongodb.ObjectId(id.trim()) },
    function (err, result) {
      if (err) throw err;
      res.send(result);
    }
  );
});

app.post("/api/v3/app/events", upload.array("files", 10), (req, res, next) => {
  const event = req.body;
  db.collection("events").insertOne(
    {
      ...event,
      files: req.files,
    },
    function (err, result) {
      if (err) throw err;
      res.send(result);
    }
  );
});
