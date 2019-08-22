# GQT-Lambda-Function
An AWS Lambda function for executing serverless GQT queries through a REST API

### Overview
Genetic Query Tools ([GQT](https://github.com/ryanlayer/gqt "GQT")) is command-line software that can be used to query large-scale genotype data. This repository contains the scripts for deploying a serverless version of that software that can be accesssed with a REST API. I recommend familiarizing yourself with using that before trying to use this serverless application of it. 

The file named 'gqt' is a Lambda function to be accessed by an API and can respond to a user with the URL of the GQT query. This URL can then be executed on a command line with curl, wget, etc. or entered in a browser search bar to download the results. The Lambda function named 'gqt_helper' executes the actual query. 'gqt' calls 'gqt_helper' and quickly responds to the API with the URL of the S3 object that 'gqt_helper' will generate once it has finished querying. This way, 'gqt' quickly responds to the user and avoids the 29 second time limit for API responses while 'gqt_helper' does the actual query in the background over the course of a couple minutes. 'GQT-Layer.zip' is an AWS Lambda layer that contains the GQT software, and 'AWSCLI-Layer.zip' is an AWS Lambda layer that contains the 'AWSCLI' package. AWSCLI is used by 'gqt_helper' to pipe the output of GQT directly to S3 to save memory for the Lambda function.

### Installation
This simply contains the code for this project, it is not designed to work in its current state. The Lambda layers can be downloaded if you want to adjust them, but public working versions of them are published and can be attached to any AWS Lambda function. 

The arn for the GQT layer is: 
```
arn:aws:lambda:us-east-1:783027271111:layer:gqt:2 
```
The arn for the AWSCLI layer is:
```
arn:aws:lambda:us-east-1:783027271111:layer:awscli:1
```
To get the rest working, you can copy and paste the code from the two Lambda functions into blank Python2.7 Lambda functions, or pull them from here and deploy them with AWS CLI. Then, attach the two Lambda layers to 'gqt_helper' using the arns above. I would recommend setting the memory for 'gqt_helper' at 3008 MB to allow for as large and as quick of a query as possible. A REST API can be created for 'gqt' in API Gateway. Create the REST API, then create a 'GET' method for it and enter 'gqt' as the Lambda function. Then, under 'Integration Request', go to 'Mapping Templates' and add a mapping template of type 'application/json'. This is the code I applied to that to allow the API to accept the correct parameters:
```
{
    "queryStringParameters": {
        #set($i = $input.params('-i'))
        #if($i && $i.length() != 0)
            "i": "$input.params('-i')",
        #else
            "i": "",
        #end
        #set($d = $input.params('-d'))
        #if($d && $d.length() != 0) 
            "d": "$input.params('-d')",
        #else
            "d": "",
        #end
        #set($c = $input.params('-c'))
        #if($c && $c.length() != 0) 
            "c": "$input.params('-c')",
        #else
            "c": "",
        #end
        #set($v = $input.params('-v'))
        #if($v && $v.length() != 0) 
            "v": "$input.params('-v')",
        #else
            "v": "",
        #end
        #set($t = $input.params('-t'))
        #if($t && $t.length() != 0) 
            "t": "$input.params('-t')",
        #else
            "t": "",
        #end
        #set($B = $input.params('-B'))
        #if($B && $B.length() != 0) 
            "B": "$input.params('-B')",
        #else
            "B": "",
        #end
        #set($V = $input.params('-V'))
        #if($V && $V.length() != 0) 
            "V": "$input.params('-V')",
        #else
            "V": "",
        #end
        #set($O = $input.params('-O'))
        #if($O && $O.length() != 0) 
            "O": "$input.params('-O')",
        #else
            "O": "",
        #end
        #set($G = $input.params('-G'))
        #if($G && $G.length() != 0) 
            "G": "$input.params('-G')",
        #else
            "G": "",
        #end
        #set($params = $input.params('params'))
        #if($params && $params.length() != 0)
            "params": "$input.params('params')"
        #else
            "params": "[]"
        #end
    }
}
```
Lastly, you need to go back to the 'GET' method's main page, into the 'Method Response' page, and add a response for 200.

With this API setup, the API doesn’t throw an error for trying to find a variable in the URL that isn’t there because if it isn't entered, it is set as an empty string. Your function doesn’t throw and error, either, for you trying to access a key in the events dictionary that doesn’t exist. 

### Usage
This API is designed to closely resemble the original [GQT query options](https://github.com/ryanlayer/gqt "GQT"). The function is deployed as described above as a REST API at https://0r1p4b2yf7.execute-api.us-east-1.amazonaws.com/Prod. An example query is 
```
https://0r1p4b2yf7.execute-api.us-east-1.amazonaws.com/Prod?-i=http://gqt-files.s3.amazonaws.com/ALL.chr1.phase3_shapeit2_mvncall_integrated_v5a.20130502.genotypes.bcf.gqt&params=[-p/Gender = 2/-g/maf()<.01/-p/Population in ('GBR','FIN','YRI')/-g/count(HET)>50]&-d=http://gqt-files.s3.amazonaws.com/gqt_v1_integrated_call_samples.20130502.ALL.ped.db
```
This queries chromosome 1 of the 1000 Genomes Project Phase 3 using files stored serverlessly in S3 to save memory. A list of all the files availible for querying can be found in 'files.txt'. The files there can also be used for local GQT queiries, as well, substatuting the files stored in S3 for local ones, or you could download the files to your local system using their URIs. All of the autosomal chromosomes of the 1000 Genomes Project Phase 3 are there. You can also use your own serverless files if they can be curled, but if a query is too large, it might exceed AWS Lambdas 3 GB limit for runtime memory. The database file that coresponds to those index files is http://gqt-files.s3.amazonaws.com/gqt_v1_integrated_call_samples.20130502.ALL.ped.db (denoted by the '-d' parameter). You can simplify the query if the files are in the 'files.txt' list by writing 'ch#' instead the URL for an index file and if you don't specify a database file, it is assumed to be the one above. So this can be shortened to:
```
https://0r1p4b2yf7.execute-api.us-east-1.amazonaws.com/Prod?-i=ch1&params=[-p/Gender = 2/-g/maf()<.01/-p/Population in ('GBR','FIN','YRI')/-g/count(HET)>50]
```
Any of the [GQT query options](https://github.com/ryanlayer/gqt "GQT") can be used with this API.

### Possible errors
Anything incorrect about the query parameters and any errors that GQT raises will either be redirected straight to the API or will be returned in the results files. The only errors I have encountered with this if it is correctly used are 1) if you try to query your own files and they are too large for AWS Lambda, or 2) if it breaks its connection with S3 before it has finished getting your query files, resulting in a 'Connection reset by peer' error. For the second error, it typically works if you retry the query, but the Lambda function is designed to automatically retry a few times if this happens.

