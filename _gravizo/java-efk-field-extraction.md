image1
skinparam componentStyle uml2;
folder "Docker Runtime" {
  interface fdld as "fluentd logging driver";
  node "Docker Container" {
    [Spring Boot app] -> stdout : step 1;
  };
  stdout -right-> fdld : step 2;
};
folder "Local Logging docker-compose" {
  cloud "Fluentd" {
    ingest -right-> [parser] : step 4;
  };  
  fdld -right-> ingest : step 3;
  database "ElasticSearch" {
    [ES index];
  };
  [parser] -up-> [ES index] : step 5;
  [Kibana] -down-> [ES index] : step 6;
};
image1
