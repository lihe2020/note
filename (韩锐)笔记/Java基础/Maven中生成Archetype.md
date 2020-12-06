#从项目创建骨架
mvn clean archetype:create-from-project
#安装骨架到本地仓库
mvn clean install -f target/generated-sources/archetype/pom.xml
mvn archetype:generate -DarchetypeCatalog=local

mvn install:install-file                      \
-Dfile=./template-archetype-1.0.0.jar     \
-DgroupId=com.yiche.backend               \
-DartifactId=template-archetype                  \
-Dversion=1.0.0                             \
-Dpackaging=jar                               \
-DgeneratePom=true

mvn archetype:generate                          \
  -DarchetypeCatalog=local                      \
  -DarchetypeGroupId=com.yiche.backend        \
  -DarchetypeArtifactId=template-archetype \
  -DgroupId=com.yiche                     \
  -DartifactId=cool                \
  -Dpackage=com.yiche.cool                      \
  -Dversion=1.0