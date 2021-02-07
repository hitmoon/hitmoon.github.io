# another way
Just for the kicks, here is another scenario, where the project to be indexed is not part of the configuration yet, so let's add it via https://github.com/oracle/opengrok/wiki/Web-services first:

curl -X POST -H Content-Type:text/plain --data bar \
    http://localhost:8080/source/api/v1/projects
and save the configuration so that it can be then used by the indexer:

curl -o new.xml http://localhost:8080/source/api/v1/configuration
cp /var/opengrok/etc/configuration.xml /var/opengrok/etc/configuration.xml.orig
mv new.xml /var/opengrok/etc/configuration.xml
This is assuming you are not using https://github.com/oracle/opengrok/wiki/Read-only-configuration, otherwise it would be necessary to merge.

Now index the project:

java -jar ~/opengrok-1.1/lib/opengrok.jar -- \
    -R /var/opengrok/etc/configuration.xml -U http://localhost:8080/source \
    bar
As you can see, no trace of Python scripts at all.