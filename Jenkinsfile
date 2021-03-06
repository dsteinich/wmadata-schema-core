pipeline {
  agent {
    node {
      label 'project:any'
    }
  }
  stages {
    stage('Set Build Description') {
      steps {
        script {
          currentBuild.description = "Deploy to ${env.DEPLOY_STAGE}"
        }
      }
    }
    stage('Clean Workspace') {
      steps {
        cleanWs()
      }
    }
    stage('Git Clone') {
      steps {
        checkout([
            $class: 'GitSCM', 
            branches: [[name: '*/master']], 
            doGenerateSubmoduleConfigurations: false, 
            extensions: [], 
            submoduleCfg: [], 
            userRemoteConfigs: [[credentialsId: 'CIDA-Jenkins-GitHub', 
            url: 'https://github.com/NWQMC/wmadata-schema-core.git']]])
      }
    }
    stage('Download liquibase jar') {
      steps {
        sh '''mkdir $WORKSPACE/wmadata
        mkdir $WORKSPACE/wmadata/dumps
        /usr/local/bin/aws s3 cp s3://owi-common-resources/resources/InstallFiles/liquibase/liquibase-$LIQUIBASE_VERSION.tar.gz $WORKSPACE/wmadata/liquibase.tar.gz
        /usr/bin/tar xzf $WORKSPACE/wmadata/liquibase.tar.gz --overwrite -C $WORKSPACE/wmadata
        /usr/local/bin/aws s3 cp s3://owi-common-resources/resources/InstallFiles/postgres/$JDBC_JAR $WORKSPACE/wmadata/lib/$JDBC_JAR
        /usr/local/bin/aws s3 cp s3://test-scnoble/gagesii.pgdump.gz $WORKSPACE/wmadata/dumps/gagesii.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/huc08.pgdump.gz $WORKSPACE/wmadata/dumps/huc08.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/huc12.pgdump.gz $WORKSPACE/wmadata/dumps/huc12.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/huc12all.pgdump.gz $WORKSPACE/wmadata/dumps/huc12all.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/huc12agg.pgdump.gz $WORKSPACE/wmadata/dumps/huc12agg.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/nhdarea.pgdump.gz $WORKSPACE/wmadata/dumps/nhdarea.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/nhdflowline_network.pgdump.gz $WORKSPACE/wmadata/dumps/nhdflowline_network.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/nhdwaterbody.pgdump.gz $WORKSPACE/wmadata/dumps/nhdwaterbody.pgdump.gz
        /usr/local/bin/aws s3 cp s3://test-scnoble/catchmentsp.pgdump.gz $WORKSPACE/wmadata/dumps/catchmentsp.pgdump.gz
        ls $WORKSPACE/wmadata/dumps
        '''
      }
    }
    stage('Run liquibase') {
      steps {
        script {
          def secretsString = sh(script: '/usr/local/bin/aws ssm get-parameter --name "/aws/reference/secretsmanager/WQP-EXTERNAL-$DEPLOY_STAGE" --query "Parameter.Value" --with-decryption --output text --region "us-west-2"', returnStdout: true).trim()
          def secretsJson =  readJSON text: secretsString
          env.WMADATA_DATABASE_ADDRESS = secretsJson.DATABASE_ADDRESS
          env.WMADATA_DATABASE_NAME = secretsJson.DATABASE_NAME
          env.WMADATA_DB_OWNER_USERNAME = secretsJson.DB_OWNER_USERNAME
          env.WMADATA_DB_OWNER_PASSWORD = secretsJson.DB_OWNER_PASSWORD
          env.WMADATA_SCHEMA_NAME = secretsJson.WMADATA_SCHEMA_NAME
          env.WMADATA_SCHEMA_OWNER_USERNAME = secretsJson.WMADATA_SCHEMA_OWNER_USERNAME
          env.WMADATA_SCHEMA_OWNER_PASSWORD = secretsJson.WMADATA_SCHEMA_OWNER_PASSWORD
          env.WMADATA_DB_READ_ONLY_USERNAME = secretsJson.WMADATA_DB_READ_ONLY_USERNAME
          env.WMADATA_DB_READ_ONLY_PASSWORD = secretsJson.WMADATA_DB_READ_ONLY_PASSWORD
          env.POSTGRES_PASSWORD = secretsJson.POSTGRES_PASSWORD
          
          sh '''

            echo $WMADATA_SCHEMA_OWNER_PASSWORD > $WORKSPACE/pgpassword.txt
            echo $WMADATA_SCHEMA_OWNER_USERNAME > $WORKSPACE/schemauser.txt
            echo $WMADATA_SCHEMA_NAME > $WORKSPACE/schema.txt
            echo $WMADATA_DATABASE_ADDRESS > $WORKSPACE/dbaddress.txt
            echo $WMADATA_DATABASE_NAME > $WORKSPACE/dbname.txt

            export LIQUIBASE_HOME=$WORKSPACE/wmadata
            export LIQUIBASE_WORKSPACE_NWIS=$WORKSPACE/liquibase/changeLogs

            chmod +x $WORKSPACE/liquibase/scripts/z1_postgres_liquibase.sh
            chmod +x $WORKSPACE/liquibase/scripts/z2_wmadata_liquibase.sh
            $WORKSPACE/liquibase/scripts/z1_postgres_liquibase.sh
            $WORKSPACE/liquibase/scripts/z2_wmadata_liquibase.sh
            '''
        }
      }
    }

    stage('Ingest Data') {
      agent {
          docker{ image 'mdillon/postgis'
           args '-v $WORKSPACE:/data'
           reuseNode true
           }
      }
      steps{

        script {

          sh '''
            export PGPASSWORD=$WMADATA_SCHEMA_OWNER_PASSWORD
            for file in /data/wmadata/dumps/*.gz; do gzip -d $file; done;

            for file in $WORKSPACE/wmadata/dumps/*.pgdump
            do
            basefile=$(basename $file)
            tablename="${basefile%.*}"
            sed -i 's/public.'$tablename'/'$WMADATA_SCHEMA_NAME'.'$tablename'/g' $file
            psql -U $WMADATA_SCHEMA_OWNER_USERNAME -f $file postgresql://$WMADATA_DATABASE_ADDRESS:5432/$WMADATA_DATABASE_NAME
            done
            '''
        }
      }
    }
  }
}
