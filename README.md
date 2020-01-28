## Amazon Cognito Custom Authentication Flow

![Cognito Custom Authentication Flow](https://docs.aws.amazon.com/cognito/latest/developerguide/images/lambda-challenges2.png)

## Define Auth Challenge

```
exports.handler = (event, context) => {
  if (event.request.session.length === 0) {
    event.response.issueTokens = false;
    event.response.failAuthentication = false;
    event.response.challengeName = 'CUSTOM_CHALLENGE';
  } else if (
    event.request.session.length === 1 &&
    event.request.session[0].challengeName === 'CUSTOM_CHALLENGE' &&
    event.request.session[0].challengeResult === true
  ) {
    event.response.issueTokens = true;
    event.response.failAuthentication = false;
  } else {
    event.response.issueTokens = false;
    event.response.failAuthentication = true;
  }
  context.done(null, event);
};
```

## Create Auth Challenge

```
/* tslint:disable */
/* eslint-disable */
const AWS = require('aws-sdk');

exports.handler = (event, context, callback) => {

  //Create a random number for otp
  const challengeAnswer = Math.random()
    .toString(10)
    .substr(2, 6);
  const phoneNumber = event.request.userAttributes.phone_number;

  //sns sms
  const sns = new AWS.SNS({ region: 'us-east-1' });
  sns.publish(
    {
      Message: 'your otp: ' + challengeAnswer,
      PhoneNumber: phoneNumber,
      MessageStructure: 'string',
      MessageAttributes: {
        'AWS.SNS.SMS.SenderID': {
          DataType: 'String',
          StringValue: 'YourSenderID'
        },
        'AWS.SNS.SMS.SMSType': {
          DataType: 'String',
          StringValue: 'Transactional'
        }
      }
    },
    function(err, data) {
      if (err) {
        console.log(err.stack);
        console.log(data);
        return;
      }
      console.log(`SMS sent to ${phoneNumber} and otp = ${challengeAnswer}`);
      return data;
    }
  );

  //set return params
  event.response.privateChallengeParameters = {};
  event.response.privateChallengeParameters.answer = challengeAnswer;
  event.response.challengeMetadata = 'CUSTOM_CHALLENGE';

  callback(null, event);
};
```

## Verify Auth Challenge Response

```
exports.handler = (event, context) => {
  if (
    event.request.privateChallengeParameters.answer ===
    event.request.challengeAnswer
  ) {
    event.response.answerCorrect = true;
  } else {
    event.response.answerCorrect = false;
  }
  context.done(null, event);
};

```

## Update IAM for Lambda

Add SNS publisher permission for the lambda function

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "arn:aws:sns:us-east-1:AWS_ACCOUNT_ID:function:PROJECT_NAME*"
        }
    ]
}
```
