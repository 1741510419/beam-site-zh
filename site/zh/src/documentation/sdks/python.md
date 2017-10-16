+---
 +layout: default
 +title: "Apache Beam Python SDK"
 +permalink: /documentation/sdks/python/
 +---
 +# Apache Beam Python SDK
 +
 +The Python SDK for Apache Beam provides a simple, powerful API for building batch data processing pipelines in Python.
 +Apache Beam �ṩ��һ�ݼ�����Ҫ�Ĺ����������ݴ���ܵ���Python API.
 +
 +## Get Started with the Python SDK
 +## ��ʼ����Python SDK֮�ð�
 +
 +Get started with the [Beam Programming Guide]({{ site.baseurl }}/documentation/programming-guide) to learn the basic concepts that apply to all SDKs in Beam. Then, follow the [Beam Python SDK Quickstart]({{ site.baseurl }}/get-started/quickstart-py) to set up your Python development environment, get the Beam SDK for Python, and run an example pipeline.
 +ת�� [Beam ���ָ��]({{ site.baseurl }}/documentation/programming-guide) ��ѧϰBeam SDK�г��ֵĻ�������. ������, ��ѭ[Beam Python SDK ���ٿ�ʼ]({{ site.baseurl }}/get-started/quickstart-py) ���������Python��������,���Python��Beam SDK,����һ���ܵ������Ӱ�.
 +
 +See the [Python API Reference]({{ site.baseurl }}/documentation/sdks/pydoc/) for more information on individual APIs.
 +�鿴 [Python API ָ��]({{ site.baseurl }}/documentation/sdks/pydoc/) ����ø�����ص�API�ĸ�����Ϣ.
 +
 +## Python Type Safety
 +## Python���Ͱ�ȫ
 +
 +Python is a dynamically-typed language with no static type checking. The Beam SDK for Python uses type hints during pipeline construction and runtime to try to emulate the correctness guarantees achieved by true static typing. [Ensuring Python Type Safety]({{ site.baseurl }}/documentation/sdks/python-type-safety) walks through how to use type hints, which help you to catch potential bugs up front with the [Direct Runner]({{ site.baseurl }}/documentation/runners/direct/).
 +Python��һ�ŵ��͵ķǾ�̬����.����Beam SDKʹ�ô���༭��ʾ����֤����ܵ���ƺ�����ʱ���Է�����ȷ������ɾ�̬����. [Python���Ͱ�ȫ�ı�֤]({{ site.baseurl }}/documentation/sdks/python-type-safety)��ͨ��ʹ�ü�����ʾ���ܰ�������[ֱ������]({{ site.baseurl }}/documentation/runners/direct/)֮ǰ����Ǳ�ڵ�BUG.
 +
 +## Managing Python Pipeline Dependencies
 +## �������Python�ܵ�����
 +
 +When you run your pipeline locally, the packages that your pipeline depends on are available because they are installed on your local machine. However, when you want to run your pipeline remotely, you must make sure these dependencies are available on the remote machines. [Managing Python Pipeline Dependencies]({{ site.baseurl }}/documentation/sdks/python-pipeline-dependencies) shows you how to make your dependencies available to the remote workers.
 +���������㱾�صĹܵ�ʱ���ܵ������İ�����ʹ���������������㱾�صĻ����ϰ�װ��Ȼ����������Զ��������Ĺܵ�ʱ�������ȷ��Զ�̻�������ͬ���и����İ���[����Python�ܵ�����]({{ site.baseurl }}/documentation/sdks/python-pipeline-dependencies)��չʾ�������Զ�̽ڵ���ȷ�����������.
 +
 +## Creating New Sources and Sinks
 +## �����µ�Դ��ͳ�
 +
 +The Beam SDK for Python provides an extensible API that you can use to create new data sources and sinks. [Creating New Sources and Sinks with the Python SDK]({{ site.baseurl }}/documentation/sdks/python-custom-io) shows how to create new sources and sinks using [Beam's Source and Sink API](https://github.com/apache/beam/blob/master/sdks/python/apache_beam/io/iobase.py).
 +Beam SDK for Pyhton�ṩ��һ������չ��API�����ڴ����µ�Դ��ͳ�.[��Python SDK�����µ�Դ��ͳ�]({{ site.baseurl }}/documentation/sdks/python-custom-io) չʾ�����ʹ�� [Beam��Դ��ͳ�API](https://github.com/apache/beam/blob/master/sdks/python/apache_beam/io/iobase.py)�����µ�Դ��ͳ�.