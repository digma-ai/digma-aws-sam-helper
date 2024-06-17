# digma-aws-sam-helper

This repository contains an example of an instrumented pet-clinic java spring application, defined in an AWS SAM template.

## How to run the example app
1. Install AWS SAM CLI: 

   https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html

2. Navigate to the `example-app` folder and run:
   ``` bash
   sam build && sam deploy --guided
   ```

## How to instrument a java application
Follow this step in order to instrument your java application defined in an AWS SAM template.
1. Add an EFS File System
   ``` yaml
      # Amazon EFS File System
      OtlpEFSFileSystem:
        Type: AWS::EFS::FileSystem
        Properties:
          FileSystemTags:
            - Key: Name
              Value: OtlpEFSFileSystem
    ```
   
2. Mount the EFS it to the subnets
   ``` yaml
      # EFS Mount Target
      OtlpEFSMountTarget1:
        Type: AWS::EFS::MountTarget
        Properties:
          FileSystemId: !Ref OtlpEFSFileSystem
          SubnetId: !Ref PublicSubnet1
          SecurityGroups: 
            - !Ref ECSSecurityGroup  
    
      OtlpEFSMountTarget2:
        Type: AWS::EFS::MountTarget
        Properties:
          FileSystemId: !Ref OtlpEFSFileSystem
          SubnetId: !Ref PublicSubnet2
          SecurityGroups: 
            - !Ref ECSSecurityGroup
    ```

3. Add the EFS as a valume to the `ECSTaskDefinition`:
   ``` yaml
   ECSTaskDefinition:
   ...
      Volumes:
        - Name: otlp-volume
          EFSVolumeConfiguration:
            FilesystemId: !Ref OtlpEFSFileSystem
    ```

4. Modify the `ECSSecurityGroup` to enable access to the EFS:
   ``` yaml
   ECSSecurityGroup:
     ...
     Properties:
       ...
       SecurityGroupIngress:
         - ...
         - IpProtocol: 'tcp'
           FromPort: 2049
           ToPort: 2049
           CidrIp: '0.0.0.0/0'
    ``` 

5. Add the java-agent-initializer task to the `ECSTaskDefinition`:
   ``` yaml
   ECSTaskDefinition:
   ...
        - Name: java-agent-initializer
          Image: digmaai/java-agent-initializer:0.0.2
          Essential: false
          LogConfiguration:
            LogDriver: 'awslogs'
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref 'AWS::Region'
              awslogs-stream-prefix: 'ecs'
          MountPoints:
            - ContainerPath: /shared-vol
              SourceVolume: otlp-volume
    ```

6. Modify task you want to instrument:
   
   Don't forget to replace the `<DIGMA-COLLECTOR-URL>` and `<ENV-NAME>` placeholders.
   ``` yaml
   ECSTaskDefinition:
   ...
        - Name: (some-java-app-to-instrument)
          ...
          DependsOn:
            - ContainerName: java-agent-initializer
              Condition: SUCCESS
          Environment:
            - Name: JAVA_TOOL_OPTIONS 
              Value: >-
                -javaagent:/shared-vol/dig-agent.jar 
                -javaagent:/shared-vol/otel-agent.jar 
                -Dotel.javaagent.extensions=/shared-vol/digma-ext.jar 
                -Dotel.exporter.otlp.traces.endpoint=<DIGMA-COLLECTOR-URL>
                -Dotel.traces.exporter=otlp 
                -Dotel.metrics.exporter=none 
                -Dotel.logs.exporter=none 
                -Dotel.exporter.otlp.protocol=grpc 
                -Dotel.instrumentation.common.experimental.controller.telemetry.enabled=true 
                -Dotel.instrumentation.common.experimental.view.telemetry.enabled=true 
                -Dotel.instrumentation.experimental.span-suppression-strategy=none 
                -Dotel.instrumentation.jdbc-datasource.enabled=true
            - Name: OTEL_SERVICE_NAME
              Value: pet-clinic
            - Name: OTEL_RESOURCE_ATTRIBUTES
              Value: digma.environment=<ENV-NAME>,digma.environment.type=Public
            # - Name: DIGMA_AUTOINSTRUMENT_PACKAGES
            #   Value: org.springframework
          MountPoints:
            - ContainerPath: /shared-vol
              SourceVolume: otlp-volume
    ```



Deploy example app:
```
sam build && sam deploy
```